#!/usr/bin/env groovy

podTemplate(containers: [
    containerTemplate(name: 'dotnet', image: 'mcr.microsoft.com/dotnet/core/sdk:3.1-alpine', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'openjdk', image: 'openjdk:11', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'node', image: 'node:12-alpine', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'docker', image: 'docker', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'kubectl', image: 'gcr.io/cloud-builders/kubectl', ttyEnabled: true, command: 'cat')
    ],
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
    ) { 
        node(POD_LABEL) {
            def docker_host = '10.1.10.11:2376'
            try{
                telegramSend (message: "Start ``` ${BRANCH_NAME} ```", chatId: -395741836)
                def stack = null
                def service = null
                switch("${BRANCH_NAME}"){
                    case ~/^.*api_catalog.*/:
                        service = 'api_catalog'
                        break
                    case ~/^.*site_foxtrot.*/:
                        service = 'site'
                        break
                    case ~/^.*api_cms.*/:
                        service = 'api_cms'
                        break
                    case ~/^.*api_exchange.*/:
                        service = 'api_exchange'
                        break
                    case ~/^.*api_main.*/:
                        service = 'api_main'
                        break
                    case ~/^.*cron.*/:
                        service = 'cron'
                        break
                }
                if ((env.BRANCH_NAME).startsWith('stage')) {
                    stack = 'stage'
                }
                if ((env.BRANCH_NAME).startsWith('release')) {
                    stack = 'release'
                }

                stage('checkout'){
                    checkout scm
                    GIT_COMMIT = sh (script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
                stage('building api'){
                    telegramSend (message: "building api ``` ${BRANCH_NAME} ```", chatId: -395741836)
                    container('dotnet'){
                        sh """
                        cd foxtrot.api
                        dotnet restore
                        dotnet build foxtrot.api.sln
                        cd ../foxtrot.models
                        cp foxtrot.models/foxtrot.models.csproj .
                        dotnet restore
                        dotnet build foxtrot.models.sln
                        cd ../foxtrot.site/foxtrot.site
                        dotnet restore
                        dotnet build ../foxtrot.site.sln
                        """
                    }
                }
                stage('building front'){
                    telegramSend (message: "building front ``` ${BRANCH_NAME} ``` ", chatId: -395741836)
                    container('node'){
                        sh """
                        cd foxtrot.site/foxtrot.site/wwwroot
                        npm install
                        npm install --global gulp-cli
                        gulp build
                        """
                    }
                    container('dotnet'){
                        sh """
                        cd foxtrot.site
                        dotnet restore
                        dotnet build foxtrot.site.sln
                        """
                    }
                }

// If merge occurred from branch stage_catalog_api, make the build and deploy catalog_api to stage environment

                if (service == 'api_catalog'){
                    container('dotnet'){
                        stage('publish api-catalog'){
                            telegramSend (message: "publish api-catalog ``` ${stack} ```", chatId: -395741836)
                            sh """
                            dotnet publish foxtrot.api/foxtrot.api-catalog/foxtrot.api-catalog.csproj -c Release -o publish
                            """
                        }
                    }
                    container('docker'){
                        stage('building image foxtrot-api-catalog') {
                            telegramSend (message: "building image foxtrot-api-catalog", chatId: -395741836)
                            dockerImageApiCatalogStage = docker.build("evinent/foxtrot-api-catalog", "--label git-commit-id=$GIT_COMMIT --label buildId=${env.BUILD_NUMBER} --label git-branch=${BRANCH_NAME} -f foxtrot.api/foxtrot.api-catalog/docker/Dockerfile .")
                        }
                        stage ('push image foxtrot-api-catalog') {
                            telegramSend (message: "push image foxtrot-api-catalog", chatId: -395741836)
                            docker.withRegistry('http://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                dockerImageApiCatalogStage.push("${BRANCH_NAME}")
                                dockerImageApiCatalogStage.push("${stack}")
                            }
                        }
                        stage('deploy to swarm'){
                            telegramSend (message: "Deploing Api-Catalog ``` ${stack} ``` -> Swarm", chatId: -395741836)
                            withDockerServer([uri: docker_host ]) {
                                docker.withRegistry('https://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
//                                    sh "tag=${BRANCH_NAME} docker stack deploy -c docker-compose-stage.yml --resolve-image always --with-registry-auth ${stack}"
                                    sh "docker service update ${stack}_${service} --image registry.foxtrot.ua/evinent/foxtrot-api-catalog:${BRANCH_NAME}"
                                }
                            }
                        }
                    }
                }

// If merge occurred from branch stage_site_foxtrot, make the build and deploy site_foxtrot to stage environment

                if (service == 'site'){
                    container('dotnet'){
                        stage('publish site_foxtrot'){
                            telegramSend (message: "publish site_foxtrot -> ``` ${stack} ``` ", chatId: -395741836)
                            sh """
                            dotnet publish foxtrot.site/foxtrot.site/foxtrot.site.csproj -c Release -o publish --no-restore
                            """
                        }
                    }
                    container('docker'){
                        stage('build image foxtrot.site') {
                            telegramSend (message: "Building image foxtrot_site -> ``` ${stack} ``` ", chatId: -395741836)
                            dockerImageSiteStage = docker.build("evinent/foxtrot-site", "--label git-commit-id=$GIT_COMMIT --label buildId=${env.BUILD_NUMBER} --label git-branch=${BRANCH_NAME} -f foxtrot.site/foxtrot.site/docker/Dockerfile .")
                        }
                        stage ('push image foxtrot.site') {
                            telegramSend (message: "push image foxtrot_site -> ``` ${stack} ``` ", chatId: -395741836)
                            docker.withRegistry('http://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                dockerImageSiteStage.push("${BRANCH_NAME}")
                                dockerImageSiteStage.push("stack")
                            }
                        }
                        stage('deploy to swarm'){
                            telegramSend (message: "Deploying site to swarm to ``` ${stack} ``` env.", chatId: -395741836)
                            withDockerServer([uri: docker_host ]) {
                                docker.withRegistry('https://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                    sh "docker service update ${stack}_${service} --image registry.foxtrot.ua/evinent/foxtrot-site:${BRANCH_NAME}"
                                }
                            }
                        }
                    }
                }

// If merge occurred from branch stage_api_cms, make the build and deploy api_cms to stage environment
                
                if (service == 'api_cms'){
                    container('dotnet'){
                        stage('publish api-catalog'){
                            telegramSend (message: "publish foxtrot.api-cms -> ``` ${stack} ``` ", chatId: -395741836)
                            sh """
                            dotnet publish foxtrot.api/foxtrot.api-cms/foxtrot.api-cms.csproj -c Release -o publish
                            """
                        }
                    }
                    container('docker'){
                        stage('build image foxtrot-api-cms') {
                            telegramSend (message: "Building image foxtrot-api-cms -> ``` ${stack} ``` ", chatId: -395741836)
                            dockerImageApiCMSStage = docker.build("evinent/foxtrot-api-cms", "--label git-commit-id=$GIT_COMMIT --label buildId=${env.BUILD_NUMBER} --label git-branch=${BRANCH_NAME} -f foxtrot.api/foxtrot.api-cms/docker/Dockerfile .")
                        }
                        stage ('push image foxtrot-api-cms') {
                            telegramSend (message: "push image foxtrot-api-cms", chatId: -395741836)
                            docker.withRegistry('http://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                dockerImageApiCMSStage.push("${BRANCH_NAME}")
                                dockerImageApiCMSStage.push("${stack}")
                            }
                        }
                        stage('deploy to swarm'){
                            telegramSend (message: "Deploying foxtrot-api-cms to swarm -> ``` ${stack} ``` env.", chatId: -395741836)
                            withDockerServer([uri: docker_host ]) {
                                docker.withRegistry('https://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
//                                    sh "tag=${BRANCH_NAME} docker stack deploy -c docker-compose-stage.yml --resolve-image always --with-registry-auth makaron_stage"
                                    sh "docker service update ${stack}_${service} --image registry.foxtrot.ua/evinent/foxtrot-api-cms:${BRANCH_NAME}"
                                }
                            }
                        }
                    }
                }

// If merge occurred from branch stage_api_exchange, make the build and deploy api_exchange to stage environment

                if (service == 'api_exchange'){
                    container('dotnet'){
                        stage('publish stage_api_exchange for ${stack}'){
                            telegramSend (message: "publish foxtrot-api-exchange ``` ${stack}``` .env.", chatId: -395741836)
                            sh """
                            dotnet publish foxtrot.api/foxtrot.api-exchange/foxtrot.api-exchange.csproj -c Release -o publish
                            """
                        }
                    }
                    container('docker'){
                        stage('building image foxtrot-api-exchange') {
                            telegramSend (message: "Building image foxtrot-api-exchange ``` ${stack}```.env.", chatId: -395741836)
                            dockerImageApiExchangeStage = docker.build("evinent/foxtrot-api-exchange", "--label git-commit-id=$GIT_COMMIT --label buildId=${env.BUILD_NUMBER} --label git-branch=${BRANCH_NAME} -f foxtrot.api/foxtrot.api-exchange/docker/Dockerfile .")
                        }
                        stage ('push image foxtrot-api-exchange') {
                            telegramSend (message: "push image foxtrot-api-exchange for ``` ${stack} ```", chatId: -395741836)
                            docker.withRegistry('http://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                dockerImageApiExchangeStage.push("${BRANCH_NAME}")
                                dockerImageApiExchangeStage.push("${stack}")
                            }
                        }
                        stage('deploy to swarm'){
                            telegramSend (message: "Deploying foxtrot-api-exchange to DockerSwarm -> ``` ${stack} ``` env.", chatId: -395741836)
                            withDockerServer([uri: docker_host ]) {
                                docker.withRegistry('https://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
//                                    sh "tag=${BRANCH_NAME} docker stack deploy -c docker-compose-stage.yml --resolve-image always --with-registry-auth makaron_stage"
                                    sh "docker service update ${stack}_${service} --image registry.foxtrot.ua/evinent/foxtrot-api-exchange:${BRANCH_NAME}"
                                }
                            }
                        }
                    }
                }

// If merge occurred from branch stage_api_main, make the build and deploy api_main to stage environment

                if (service == 'api_main'){
                    container('dotnet'){
                        stage('publish stage_api_main'){
                            telegramSend (message: "publish foxtrot-api-main ``` ${stack}```.env.", chatId: -395741836)
                            sh """
//                            dotnet publish foxtrot.api/foxtrot.api-main/foxtrot.api-main.csproj -c Release -o publish --no-restore  /p:ShowLinkerSizeComparison=true
                            dotnet publish foxtrot.api/foxtrot.api-main/foxtrot.api-main.csproj -c Release -o publish
                            """
                        }
                    }
                    container('docker'){
                        stage('building image foxtrot-api_main for ${stack}.env.') {
                            telegramSend (message: "Building image foxtrot-api-main ```${stack}```", chatId: -395741836)
                            dockerImageApiMainStage = docker.build("evinent/foxtrot-api-main", "--label git-commit-id=$GIT_COMMIT --label buildId=${env.BUILD_NUMBER} --label git-branch=${BRANCH_NAME} -f foxtrot.api/foxtrot.api-main/docker/Dockerfile .")
                        }
                        stage ('push image foxtrot-api-main -> ${stack}.env.') {
                            telegramSend (message: "Push image foxtrot-api-main", chatId: -395741836)
                            docker.withRegistry('http://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                dockerImageApiMainStage.push("${BRANCH_NAME}")
                                dockerImageApiMainStage.push("${stack}")
                            }
                        }
                        stage('deploy to swarm'){
                            telegramSend (message: "Deploy foxtrot-api-main to DockerSwarm -> ``` ${stack}``` env.", chatId: -395741836)
                            withDockerServer([uri: docker_host ]) {
                                docker.withRegistry('https://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
//                                    sh "tag=${BRANCH_NAME} docker stack deploy -c docker-compose-stage.yml --resolve-image always --with-registry-auth makaron_stage"
                                    sh "docker service update ${stack}_${service} --image registry.foxtrot.ua/evinent/foxtrot-api-exchange:${BRANCH_NAME}"
                                }
                            }
                        }
                    }
                }


// If merge occurred from branch release_cron, make the build and deploy cron to production environment

                if (service == 'api_cron'){
                    container('docker'){
                        stage('build image release_cron') {
                            telegramSend (message: "Building image release_cron", chatId: -395741836)
                            dockerImageCronRelease = docker.build("evinent/foxtrot-cron", "--label git-commit-id=$GIT_COMMIT --label buildId=${env.BUILD_NUMBER} --label git-branch=${BRANCH_NAME} -f foxtrot.cron/docker/Dockerfile .")
                        }
                        stage ('push image release_cron') {
                            telegramSend (message: "push image release_cron", chatId: -395741836)
                            docker.withRegistry('http://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
                                dockerImageCronRelease.push("${BRANCH_NAME}")
                                dockerImageCronRelease.push("release")
                            }
                        }
                        stage('deploy to swarm'){
                            telegramSend (message: "Deploying Cron service to DockerSwarm -> prod.env.", chatId: -395741836)
                            withDockerServer([uri: docker_host ]) {
                                docker.withRegistry('https://registry.foxtrot.ua/', '1c1a1aac-b668-4839-9bc1-8f5bfd33759f'){
//                                    sh "tag=${BRANCH_NAME} docker stack deploy -c docker-compose-prod.yml --resolve-image always --with-registry-auth ${stack}"
                                    sh "docker service update ${stack}_${service} --image registry.foxtrot.ua/evinent/cron:${BRANCH_NAME}"
                            }
                        }
                    }
                }
            }
             
                currentResult = "SUCCESS"
            }   catch (e) {
                    currentResult = "FAILED"
                    throw e
                } finally {
                    int minutes = (int) ((currentBuild.duration / 1000) / 60);
                    telegramSend (message: "Name: ``` ${currentBuild.fullDisplayName} ``` \
                    ``` JOB Name: ${JOB_NAME} ``` \
                    [link to job](${JOB_URL}) \
                    ``` Result: ``` *${currentResult}* \
                    ``` Duration (in minutes): ${minutes} ``` ", chatId: -395741836)
            }
        }
    }
