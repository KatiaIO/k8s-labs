# Exercice Pratique : Webhook d'Admission Kubernetes

## Objectif
Créer un webhook d'admission qui :
1. Valide que tous les pods ont des limites de ressources définies
2. Ajoute automatiquement des labels de monitoring
3. Force l'utilisation d'images provenant d'un registry approuvé

## Étape 1 : Création des Certificats

```bash
# Création d'une autorité de certification
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 365 -key ca.key -out ca.crt -subj "/CN=Admission CA"

# Création des certificats pour le webhook
openssl genrsa -out webhook-tls.key 2048
openssl req -new -key webhook-tls.key -out webhook-tls.csr -subj "/CN=admission-webhook.default.svc"

# Création du fichier de configuration pour le certificat
cat > webhook-csr.conf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = admission-webhook
DNS.2 = admission-webhook.default
DNS.3 = admission-webhook.default.svc
EOF

# Signature du certificat
openssl x509 -req -in webhook-tls.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out webhook-tls.crt -days 365 \
    -extensions v3_req -extfile webhook-csr.conf
```

## Étape 2 : Création du Service et du Déploiement

```yaml
# webhook-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      containers:
      - name: webhook
        image: admission-webhook:v1  # Image à construire
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: webhook-certs
          mountPath: /etc/webhook/certs
          readOnly: true
      volumes:
      - name: webhook-certs
        secret:
          secretName: webhook-certs

---
apiVersion: v1
kind: Service
metadata:
  name: admission-webhook
spec:
  ports:
  - port: 443
    targetPort: 8443
  selector:
    app: admission-webhook
```

## Étape 3 : Implémentation du Webhook (Go)

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func validatePod(pod *corev1.Pod) (bool, string) {
    // Vérification des limites de ressources
    for _, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil {
            return false, fmt.Sprintf("Container %s doit avoir des limites de ressources", container.Name)
        }
    }

    // Vérification du registry
    for _, container := range pod.Spec.Containers {
        if !strings.HasPrefix(container.Image, "approved-registry.com/") {
            return false, "Seules les images de approved-registry.com sont autorisées"
        }
    }

    return true, ""
}

func mutatePod(pod *corev1.Pod) []patchOperation {
    var patches []patchOperation

    // Ajout des labels de monitoring
    if pod.Labels == nil {
        pod.Labels = map[string]string{}
    }
    
    patches = append(patches, patchOperation{
        Op:    "add",
        Path:  "/metadata/labels/monitoring",
        Value: "enabled",
    })

    return patches
}

func handleAdmission(w http.ResponseWriter, r *http.Request) {
    // Lecture du body
    var admissionReview admissionv1.AdmissionReview
    if err := json.NewDecoder(r.Body).Decode(&admissionReview); err != nil {
        http.Error(w, "Erreur de décodage", http.StatusBadRequest)
        return
    }

    // Désérialisation du pod
    pod := &corev1.Pod{}
    if err := json.Unmarshal(admissionReview.Request.Object.Raw, pod); err != nil {
        http.Error(w, "Erreur de parsing du pod", http.StatusBadRequest)
        return
    }

    // Validation
    allowed, message := validatePod(pod)
    
    // Mutation si la validation passe
    var patches []patchOperation
    if allowed {
        patches = mutatePod(pod)
    }

    // Préparation de la réponse
    response := admissionv1.AdmissionResponse{
        Allowed: allowed,
        Result: &metav1.Status{
            Message: message,
        },
    }

    // Ajout des patches si nécessaire
    if allowed && len(patches) > 0 {
        patchBytes, _ := json.Marshal(patches)
        response.Patch = patchBytes
        pt := admissionv1.PatchTypeJSONPatch
        response.PatchType = &pt
    }

    // Envoi de la réponse
    admissionReview.Response = &response
    json.NewEncoder(w).Encode(admissionReview)
}
```

## Étape 4 : Configuration du Webhook dans Kubernetes

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy
webhooks:
- name: pod-policy.default.svc
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
  clientConfig:
    service:
      name: admission-webhook
      namespace: default
      path: "/validate"
    caBundle: ${CA_BUNDLE}
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-mutating
webhooks:
- name: pod-mutating.default.svc
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
  clientConfig:
    service:
      name: admission-webhook
      namespace: default
      path: "/mutate"
    caBundle: ${CA_BUNDLE}
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

## Étape 5 : Tests

### Test de Validation
```yaml
# test-pod-invalid.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: unapproved-registry.com/nginx  # Devrait être rejeté
```

### Test de Mutation
```yaml
# test-pod-valid.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: approved-registry.com/nginx
    resources:
      limits:
        cpu: "200m"
        memory: "256Mi"
```

## Vérification
```bash
# Test du pod invalide
kubectl apply -f test-pod-invalid.yaml

# Test du pod valide
kubectl apply -f test-pod-valid.yaml

# Vérification des labels ajoutés
kubectl get pod test-pod -o jsonpath='{.metadata.labels}'
```