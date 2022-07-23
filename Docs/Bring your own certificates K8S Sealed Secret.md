# Bring your own certificates K8S Sealed Secret 

In our project, we have a scenario where manage all secrets Uniformly. 

for each prod cluster, we have to generate their own RSA key pair(private & public key) and then apply into cluster pulled from github repository.



The controller generates its own certificates when is deployed for the first time, it also manages the renewal for you. But you can also bring your own certificates so the controller can consume them as well.

The controller consumes certificates contained in any secret labeled with `sealedsecrets.bitnami.com/sealed-secrets-key=active`, the secret has to live in the same namespace as the controller. There can be multiple such secrets.

Below you can find all the steps needed to create and consume your own certificates.

## Set your vars

```
export PRIVATEKEY="mytls.key"
export PUBLICKEY="mytls.crt"
export NAMESPACE="sealed-secrets"
export SECRETNAME="mycustomkeys"
```

## Generate a new RSA key pair (certificates)

```
openssl req -x509 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"
```

**above script is recommend by official website , but it doesn't work to me, it only generates the private key, public key can't find in the same folder.**

therefore I update the script slightly as below, it works 

```
openssl req -x509 -newkey rsa:4096 -out client.crt -keyout client.key -batch -nodes
```

**To replace data.tls.crt & key** in the masterkey.yaml file

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    sealedsecrets.bitnami.com/sealed-secrets-key: active
  name: sealed-secrets-key
  namespace: flux-system    
type: kubernetes.io/tls
data:
    tls.crt: XXX
    tls.key: XXX
```



We can upload masterkey.yaml file to your github repository, which aims to unified manage all the secrets for different cluster.

## Create a tls k8s secret, using your recently created RSA key pair

```
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY"
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active
```

## Deleting the controller Pod is needed to pick they new keys

```
kubectl -n  "$NAMESPACE" delete pod -l name=sealed-secrets-controller
kubectl delete pod  sealed-secrets-77594b99dc-xqfhg    -n flux-system
```

## See the new certificates (private keys) in the controller logs

```
kubectl -n "$NAMESPACE" logs -l name=sealed-secrets-controller

controller version: v0.12.1+dirty
2020/06/09 14:30:45 Starting sealed-secrets controller version: v0.12.1+dirty
2020/06/09 14:30:45 Searching for existing private keys
2020/06/09 14:30:45 ----- sealed-secrets-key5rxd9
2020/06/09 14:30:45 ----- mycustomkeys
2020/06/09 14:30:45 HTTP server serving on :8080
```

## Try your own certificates

Now you can try to seal a secret with your own certificate, instead of using the controller provided ones.

### Used your recently created public key to "seal" your secret

Use your own certificate (key) by using the `--cert` flag:

```
kubeseal --cert "./${PUBLICKEY}" --scope cluster-wide < mysecret.yaml | kubectl apply -f-
or
kubeseal --format=yaml --cert="$PUBLICKEY" < ghe-https-credentials.yaml > ghe-https-credentials-sealed.yaml

kubectl apply -f  ghe-https-credentials-sealed.yaml -n "$NAMESPACE"

```

### We can see the secret has been unsealed successfully

```
kubectl -n "$NAMESPACE" logs -l name=sealed-secrets-controller

controller version: v0.12.1+dirty
2020/06/09 14:30:45 Starting sealed-secrets controller version: v0.12.1+dirty
2020/06/09 14:30:45 Searching for existing private keys
2020/06/09 14:30:45 ----- sealed-secrets-key5rxd9
2020/06/09 14:30:45 ----- mycustomkeys
2020/06/09 14:30:45 HTTP server serving on :8080
2020/06/09 14:37:55 Updating test-namespace/mysecret
2020/06/09 14:37:55 Event(v1.ObjectReference{Kind:"SealedSecret", Namespace:"test-namespace", Name:"mysecret", UID:"f3a6c537-d254-4c06-b08f-ab9548f28f5b", APIVersion:"bitnami.com/v1alpha1", ResourceVersion:"20469957", FieldPath:""}): type: 'Normal' reason: 'Unsealed' SealedSecret unsealed successfully
```

**NOTE:**

`$PRIVATEKEY` is your private key, which is used by the controller to unseal your secret. Don't share it with anyone you don't trust, and save it in a safe place!!



#### Reference links:

[sealed-secrets/bring-your-own-certificates.md at main · bitnami-labs/sealed-secrets (github.com)](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/bring-your-own-certificates.md)

[Creating Kubernetes Secrets Using TLS/SSL as an Example | shocksolution.com](https://shocksolution.com/2018/12/14/creating-kubernetes-secrets-using-tls-ssl-as-an-example/)

