# cert-manager(dsn01)

## main usage
* create certification of ssl automatically by cert-manager
* use dns01 method

## purpose
* create a kubernetes cluster by kind
* setup cert-manager
* setup alidns-webhook

## precondition
* [kind-cluster](/basics/kubernetesernetes/kind-cluster.md)

## operation
1. prepare [cert-manager.values.yaml](resources/cert-manager.values.yaml.md)
2. prepare images
    * ```shell
      DOCKER_IMAGE_PATH=/root/docker-images && mkdir -p $DOCKER_IMAGE_PATH
      BASE_URL="https://resource.cnconti.cc:32443/docker-images"
      for IMAGE in "quay.io_jetstack_cert-manager-controller_v1.5.4.dim" \
          "quay.io_jetstack_cert-manager-webhook_v1.5.4.dim" \
          "quay.io_jetstack_cert-manager-cainjector_v1.5.4.dim" \
          "quay.io_jetstack_cert-manager-ctl_v1.5.4.dim" \
          "quay.io_jetstack_cert-manager-acmesolver_v1.5.4.dim" \
          "ghcr.io_devmachine-fr_cert-manager-alidns-webhook_cert-manager-alidns-webhook_0.2.0.dim"
      do
          IMAGE_FILE=$DOCKER_IMAGE_PATH/$IMAGE
          if [ ! -f $IMAGE_FILE ]; then
              TMP_FILE=$IMAGE_FILE.tmp \
                  && curl -o "$TMP_FILE" -L "$BASE_URL/$IMAGE" \
                  && mv $TMP_FILE $IMAGE_FILE 
          fi
          docker image load -i $IMAGE_FILE && rm -f $IMAGE_FILE
      done
      DOCKER_REGISTRY="localhost:5000"
      for IMAGE in "quay.io/jetstack/cert-manager-controller:v1.5.4" \
          "quay.io/jetstack/cert-manager-webhook:v1.5.4" \
          "quay.io/jetstack/cert-manager-cainjector:v1.5.4" \
          "quay.io/jetstack/cert-manager-ctl:v1.5.4" \
          "quay.io/jetstack/cert-manager-acmesolver:v1.5.4" \
          "ghcr.io/devmachine-fr/cert-manager-alidns-webhook/cert-manager-alidns-webhook:0.2.0"
      do
          DOCKER_TARGET_IMAGE=$DOCKER_REGISTRY/$IMAGE
          docker tag $IMAGE $DOCKER_TARGET_IMAGE \
              && docker push $DOCKER_TARGET_IMAGE \
              && docker image rm $DOCKER_TARGET_IMAGE
      done
      ```
3. install `cert-manager` by helm
    * ```shell
       helm install \
           --create-namespace --namespace basic-components \
           my-cert-manager \
           https://resource.cnconti.cc:32443/charts/charts.jetstack.io/cert-manager-v1.5.4.tgz \
           --values cert-manager.values.yaml \
           --atomic
       ```
4. make sure permissions added to `$YOUR_ACCESS_KEY_ID`
    * ```json
      {
        "Version": "1",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": [
              "alidns:AddDomainRecord",
              "alidns:DeleteDomainRecord"
            ],
            "Resource": "acs:alidns:*:*:domain/cnconti.cc"
          }, {
            "Effect": "Allow",
            "Action": [
              "alidns:DescribeDomains",
              "alidns:DescribeDomainRecords"
            ],
            "Resource": "acs:alidns:*:*:domain/*"
          }
        ]
      }
      ```
5. create secret `alidns-webhook-secrets`
    * ```shell
      kubectl -n basic-components create secret generic alidns-webhook-secrets \
          --from-literal="access-token=$YOUR_ACCESS_KEY_ID" \
          --from-literal="secret-key=$YOUR_ACCESS_KEY_SECRET"
      ```
6. prepare [alidns-webhook.values.yaml](resources/alidns-webhook.values.yaml.md)
7. install `alidns-webhook` by helm
    * ```shell
      helm install \
          --create-namespace --namespace basic-components \
          my-alidns-webhook \
          https://resource.cnconti.cc:32443/charts/other/alidns-webhook-0.6.0.tgz \
          --values alidns-webhook.values.yaml \
          --atomic
      ```
8. create ClusterIssuer `alidns-webhook-letsencrypt`
    * prepare [alidns-webhook.clusterissuer.yaml](resources/alidns-webhook.clusterissuer.yaml.md)
    * ```shell
      kubectl -n basic-components apply -f alidns-webhook.clusterissuer.yaml
      ```

## test
1. TODD

## uninstall
1. delete ClusterIssuer `self-signed-cluster-issuer`
    * ```shell
      kubectl -n basic-components delete clusterissuer self-signed-cluster-issuer
      ```
2. uninstall `my-alidns-webhook`
    * ```shell
      helm -n basic-components uninstall my-alidns-webhook
      ```
3. uninstall `my-cert-manager`
    * ```shell
      helm -n basic-components uninstall my-cert-manager
      ```
