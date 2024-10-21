# BAREMETAL K8S OAKDEW

### Run in local

- Actualiza el archivo /etc/hosts
  Con este enfoque, solo necesitas una línea en tu archivo /etc/hosts:

`192.168.1.230 oakdew.local`

### Set storage class default:
kubectl patch storageclass <STORAGE_CLASS> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

### Install MetalLb

Configure `kube-proxy`:

```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

### Install NGINX Ingress Controller

```
helm upgrade --install ingress-nginx ingress-nginx \
 --repo https://kubernetes.github.io/ingress-nginx \
 --namespace ingress-nginx --create-namespace
```

You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:

```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls
```

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

```
  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

## Setup Gitlab

Update file `gitlab.rb` to configure Redis:

```
gitlab_rails['redis_host'] = 'redis'
gitlab_rails['redis_port'] = 6379
gitlab_rails['redis_ssl'] = false
gitlab_rails['redis_password'] = nil
gitlab_rails['redis_database'] = 0
gitlab_rails['redis_enable_client'] = true

...
external_url <URL_FOR_GITLABL_TO_REACH_ITSELF>

```

We can define variable `EXTERNAL_URL` when installing Gitlab to define this variable in the file.

Then restart the Gitlab:

```
gitlab-ctl reconfigure
gitlab-ctl restart
```

### 500 error when accessing CI/CD in a project

In Gitlab Pod Terminal run in `gitlab-rails console`:

```
projects = Project.all

project = Project.find(<PROJECT_ID>)

if project
  project.update(runners_token_encrypted: nil)
else
  puts "Project not found."
end
```

Same erro for CI/CD in admin:

```
gitlab-rails dbconsole

-- Clear project tokens, Clear group tokens, Clear instance tokens, Clear runner tokens
UPDATE projects SET runners_token = null, runners_token_encrypted = null;
UPDATE namespaces SET runners_token = null, runners_token_encrypted = null;
UPDATE application_settings SET runners_registration_token_encrypted = null;
UPDATE ci_runners SET token = null, token_encrypted = null;

https://docs.gitlab.com/ee/raketasks/backup_restore.html#reset-runner-registration-tokens
```

Edit:

After clearing existing pipeline jobs (see above), I could still not open the ci settings page for some migrated projects where I had set environment variables. In this case try to remove them:

```
gitlab-rails dbconsole
SELECT * FROM ci_variables;
DELETE FROM ci_variables WHERE project_id='XX';
```

### Install Gitlab runners in kubernetes

```
helm repo add gitlab https://charts.gitlab.io
helm repo update gitlab
helm search repo -l gitlab/gitlab-runner | head -n10 # Search version to install
helm pull gitlab/gitlab-runner --version <VERSION>

tar xf gitlab-runner-<VERSION>.tgz
cd gitlab-runner
```

Create a new Project Runner in Gitlab to obtain the runner token. With the runner token, install the runner in the cluster:

```
helm install --namespace devops gitlab-runner -f helm_values.yaml gitlab/gitlab-runner
```

Set external URL for Gitlab in file `/etc/gitlab/gitlab.rb `:

```
##! Note: During installation/upgrades, the value of the environment variable
##! EXTERNAL_URL will be used to populate/replace this value.
##! On AWS EC2 instances, we also attempt to fetch the public hostname/IP
##! address from AWS. For more details, see:
##! https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html

external_url "http://gitlab.example.com"
```

Set outbound conection in networks to allow webhook to local network. Get a access token in Gitlab from your user prefferences and use it to make this call:

```
curl --request PUT --header "PRIVATE-TOKEN: <TOKEN>" \
--header "Content-Type: application/json" \
--data '{
  "allow_local_requests_from_web_hooks_and_services": true
  }' \
"http://gitlab.oakdew.local/api/v4/application/settings"
```

Verify configurations:

```
curl --header "PRIVATE-TOKEN: <TOKEN>" \
"http://gitlab.oakdew.local/api/v4/application/settings"
```

When you have error 500 in setting, it can be related with error `OpenSSL::Cipher::CipherError`. Try to run this in Pod terminal:

```
gitlab-rails console

ApplicationSetting.first.delete
ApplicationSetting.first
exit
```

### Harbor from local machine
To `docker login` to harbor from local machine, you need to add the trusted insecure:

```
sudo nano /etc/docker/daemon.json
```

Add the following content:

```
{
  "insecure-registries" : ["harbor.oakdew.local]
}
```
