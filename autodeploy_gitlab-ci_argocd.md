Пример пайлайна для автодеплоя 


```
variables:
  PROJECT_NAME: onecell
  APP_NAME: department-audit

build-image:
  stage: build
  image: $DOCKER_REGISTRY_PROXY/docker:latest
  variables:
    BUILD_IMAGE_NAME: $CI_PROJECT_NAME-sha$CI_COMMIT_SHORT_SHA
    DOCKER_CONFIG: '/tmp/.docker'
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  services:
    - name: docker:dind
      command: ["--mtu=1300"]
  before_script:
    - until docker info; do sleep 1; done
  script:
    - unset CI
    - docker build -t $BUILD_IMAGE_NAME .
    - docker tag $BUILD_IMAGE_NAME "$DOCKER_REGISTRY/$PROJECT_NAME/$APP_NAME:latest"
    - docker tag $BUILD_IMAGE_NAME "$DOCKER_REGISTRY/$PROJECT_NAME/$APP_NAME:$CI_COMMIT_REF_SLUG"
    - docker tag $BUILD_IMAGE_NAME "$DOCKER_REGISTRY/$PROJECT_NAME/$APP_NAME:sha$CI_COMMIT_SHORT_SHA"
    - docker push --all-tags "$DOCKER_REGISTRY/$PROJECT_NAME/$APP_NAME"
    - docker image rm $DOCKER_REGISTRY/$PROJECT_NAME/$APP_NAME:$CI_COMMIT_REF_SLUG
    - docker image rm $DOCKER_REGISTRY/$PROJECT_NAME/$APP_NAME:sha$CI_COMMIT_SHORT_SHA
    - docker image rm $BUILD_IMAGE_NAME

deploy:
  when: manual
  variables:
    GIT_STRATEGY: none
    TAG: sha$CI_COMMIT_SHORT_SHA
    GIT_SSH_COMMAND: ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i /tmp/id_rsa
    ARGOCD_PASSWORD: gx0ElousI7Krg75K
    ARGOCD_SERVER: $PMVP_PROD_ARGOCD_SERVER
    ARGOCD_USERNAME: $PMVP_PROD_ARGOCD_USERNAME
  stage: deploy
  image: $DOCKER_REGISTRY_PROXY/argocd-deployer-image:v5
  before_script:
    - echo $GITLAB_ID_RSA | base64 -d > /tmp/id_rsa
    - chmod 400 /tmp/id_rsa
    - cat /etc/hosts
    - argocd login "$ARGOCD_SERVER" --insecure --username="$ARGOCD_USERNAME" --password="$ARGOCD_PASSWORD"
  script:
    - apk update
    - apk add --no-cache skopeo
    - skopeo --version
    - git clone git@gitlab.onecell.ru:iac/cloudai/cloudai-iac.git
    - cd cloudai-iac/charts/cluster/templates 
    - NEW_SHA=$(skopeo inspect --format '{{ .Digest }}' docker://docker.onecell.su/onecell/${APP_NAME}:sha$CI_COMMIT_SHORT_SHA)
    - echo $NEW_SHA
    - OLD_SHA=$(yq -r '.spec.source.helm.values' ${APP_NAME}.yaml | yq -r '.image.sha256')
    - echo $OLD_SHA
    - |
      if [ "$OLD_SHA" != "$NEW_SHA" ]; then
        echo "🔄 Обновляем SHA..."
        sed -i "s|$OLD_SHA|$NEW_SHA|" ${APP_NAME}.yaml
        cat ${APP_NAME}.yaml
        git add ${APP_NAME}.yaml 
        git commit -m "update from job autodeploy"
        git push

        argocd app wait argocd/cluster --operation --grpc-web --resource argoproj.io:Application:argocd/${APP_NAME}
        argocd app sync argocd/cluster --force --grpc-web --resource argoproj.io:Application:argocd/${APP_NAME}
        argocd app wait argocd/cluster --sync --grpc-web --resource argoproj.io:Application:argocd/${APP_NAME}

        argocd app wait ${APP_NAME}  --operation --grpc-web
        argocd app sync ${APP_NAME} --force --grpc-web
        argocd app wait ${APP_NAME} --sync --grpc-web
        
      else
        echo "✅ SHA актуален."
        cat ${APP_NAME}.yaml
      fi
```
