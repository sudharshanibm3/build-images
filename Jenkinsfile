library 'magic-butler-catalogue'

pipeline {
  agent none

  options {
    timestamps()
    ansiColor 'xterm'
  }
  triggers {
    issueCommentTrigger('.*test this please.*')
    cron(env.BRANCH_NAME ==~ /\d\.\d/ ? 'H H 1,15 * *' : '')
  }
  environment {
    DOCKER_BUILDKIT='1'
    SCCACHE_BUCKET='logdna-sccache-us-west-2'
    SCCACHE_REGION='us-west-2'
  }
  parameters {
    booleanParam(name: 'PUBLISH_IMAGE', description: 'Publish docker image to Google Container Registry (GCR)', defaultValue: false)
  }
  stages {
    stage('Validate PR Source') {
      when {
        expression { env.CHANGE_FORK }
        not {
            triggeredBy 'issueCommentCause'
        }
      }
      steps {
        error("A maintainer needs to approve this PR with a comment of '${TRIGGER_STRING}'")
      }
    }

    stage('Build base images for each supported build host platform') {
      matrix {
        axes {
          axis {
            name 'RUSTC_VERSION'
            values 'stable', 'beta'
          }
          axis {
            name 'VARIANT_VERSION'
            values 'buster', 'bullseye'
          }
          // Host architecture of the built image
          axis {
            name 'PLATFORM'
            // Support for x86_64 and arm64 for Mac M1s/AWS graviton devs/builders
            values 'linux/amd64', 'linux/arm64'
          }
        }
        agent {
          node {
            label 'ec2-fleet'
            customWorkspace "docker-images-${BUILD_NUMBER}"
          }
        }
        stages {
          stage('Initilize qemu') {
            steps {
              sh """
                free -h && df -h
                cat /proc/cpuinfo
                # initialize qemu
                docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
              """
            }
          }
          stage('Build platform specific base') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]){
                    sh """
                        echo "[default]" > ${WORKSPACE}/.aws_creds
                        echo 'AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID' >> ${WORKSPACE}/.aws_creds
                        echo 'AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY' >> ${WORKSPACE}/.aws_creds
                    """
                    script {
                        def image_name = generateImageName(
                          name: "rust"
                          , variant_base: "debian"
                          , variant_version: "${VARIANT_VERSION}"
                          , version: "${RUSTC_VERSION}"
                          , image_suffix: "base-${PLATFORM.replaceAll('/','-')}"
                        )

                        buildImage(
                          name: "rust"
                          , variant_base: "debian"
                          , variant_version: "${VARIANT_VERSION}"
                          , version: "${RUSTC_VERSION}"
                          , platform: "${PLATFORM}"
                          , dockerfile: "Dockerfile.base"
                          , image_name: image_name
                          , pull: true
                          , push: true
                          , clean: false
                        )
                    }
                }
            }
            post {
                always {
                    sh "rm ${WORKSPACE}/.aws_creds"
                }
            }
          } // End Build stage
        } // End Build Rust Images stages
      } // End matrix
    } // End Build Rust Images stage
    // Build the images containing the cross compilers targeting actual
    // distribution platforms
    stage('Build CROSS_COMPILER_TARGET_ARCH Specific images on top of PLATFORM's base image') {
      matrix {
        axes {
          axis {
            name 'RUSTC_VERSION'
            values 'stable', 'beta'
          }
          axis {
            name 'VARIANT_VERSION'
            values 'buster', 'bullseye'
          }
          // Target ISA for musl cross comp toolchain and precompiled libs
          axis {
            name 'CROSS_COMPILER_TARGET_ARCH'
            values 'x86_64', 'aarch64'
          }
          // Host architecture of the built image
          axis {
            name 'PLATFORM'
            values 'linux/amd64', 'linux/arm64'
          }
        }
        agent {
          node {
            label 'ec2-fleet'
            customWorkspace "docker-images-${BUILD_NUMBER}"
          }
        }
        stages {
          stage('Initilize qemu') {
            steps {
              sh """
                free -h && df -h
                cat /proc/cpuinfo
                # initialize qemu
                docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
              """
            }
          }
          stage('Build cross compilation image') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]){
                    sh """
                        echo "[default]" > ${WORKSPACE}/.aws_creds
                        echo 'AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID' >> ${WORKSPACE}/.aws_creds
                        echo 'AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY' >> ${WORKSPACE}/.aws_creds
                    """
                    script {
                      def image_name = generateImageName(
                        name: "rust"
                        , variant_base: "debian"
                        , variant_version: "${VARIANT_VERSION}"
                        , version: "${RUSTC_VERSION}"
                        , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}-${PLATFORM.replaceAll('/','-')}"
                      )
                      def base_name = generateImageName(
                        name: "rust"
                        , variant_base: "debian"
                        , variant_version: "${VARIANT_VERSION}"
                        , version: "${RUSTC_VERSION}"
                        , image_suffix: "base-${CROSS_COMPILER_TARGET_ARCH}-${PLATFORM.replaceAll('/','-')}"
                      )
                      // GCR image
                      buildImage(
                        name: "rust"
                        , variant_base: "debian"
                        , variant_version: "${VARIANT_VERSION}"
                        , version: "${RUSTC_VERSION}"
                        , cross_compiler_target_arch: "${CROSS_COMPILER_TARGET_ARCH}"
                        , platform: "${PLATFORM}"
                        , dockerfile: "Dockerfile"
                        , image_name: image_name
                        , base_name: base_name
                        , pull: true
                        , clean: false
                      )
                      def docker_name = generateImageName(
                        repo_base: "docker.io/logdna",
                        , name: "build-images"
                        , variant_base: "rust"
                        , variant_version: "rust-${VARIANT_VERSION}"
                        , version: "${RUSTC_VERSION}"
                        , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}-${PLATFORM.replaceAll('/','-')}"
                      )
                      // Dockerhub image
                      docker.withRegistry(
                                'https://index.docker.io/v1/',
                                'dockerhub-username-password') {
                            buildImage(
                                repo_base: "docker.io/logdna",
                                , name: "rust"
                                , variant_base: "debian"
                                , variant_version: "${VARIANT_VERSION}"
                                , version: "${RUSTC_VERSION}"
                                , cross_compiler_target_arch: "${CROSS_COMPILER_TARGET_ARCH}"
                                , platform: "${PLATFORM}"
                                , dockerfile: "Dockerfile"
                                , image_name: docker_name
                                , base_name: base_name
                            )
                      }
                      try {
                        gcr.clean(base_name)
                        // Hack to work around docker image bug
                        gcr.clean(image_name.replaceFirst("docker.io/", ""))
                      } catch(Exception ex) {
                        println("image already cleaned up");
                      }
                    }
                }
            }
            post {
                always {
                    sh "rm ${WORKSPACE}/.aws_creds"
                }
            }
          } // End Build stage
        } // End Build Rust Images stages
      }
    }
    stage('Create Multi-Arch Manifests/Images') {
      matrix {
        axes {
          axis {
            name 'RUSTC_VERSION'
            values 'stable', 'beta'
          }
          axis {
            name 'VARIANT_VERSION'
            values 'buster', 'bullseye'
          }
          // Target ISA for musl cross comp toolchain and precompiled libs
          axis {
            name 'CROSS_COMPILER_TARGET_ARCH'
            values 'x86_64', 'aarch64'
          }
        }
        agent {
          node {
            label 'ec2-fleet'
            customWorkspace "docker-images-${BUILD_NUMBER}"
          }
        }
        stages {
          stage ('Create Multi Arch Manifest') {
            when {
                expression { return ((env.CHANGE_BRANCH  == "main" || env.BRANCH_NAME == "main" ) || env.PUBLISH_IMAGE) }
            }
            steps {
              script {
                def gcr_image_name = generateImageName(
                   name: "rust"
                   , variant_base: "debian"
                   , variant_version: "${VARIANT_VERSION}"
                   , version: "${RUSTC_VERSION}"
                   , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}"
                   )
                def gcr_amd64_image_name = generateImageName(
                   name: "rust"
                   , variant_base: "debian"
                   , variant_version: "${VARIANT_VERSION}"
                   , version: "${RUSTC_VERSION}"
                   , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}-amd64"
                   )
                def gcr_arm64_image_name = generateImageName(
                   name: "rust"
                   , variant_base: "debian"
                   , variant_version: "${VARIANT_VERSION}"
                   , version: "${RUSTC_VERSION}"
                   , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}-arm64"
                   )
                def docker_image_name = generateImageName(
                    repo_base: "docker.io/logdna",
                    , name: "build-images"
                    , variant_base: "rust"
                    , variant_version: "rust-${VARIANT_VERSION}"
                    , version: "${RUSTC_VERSION}"
                    , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}"
                    )
                def docker_amd64_image_name = generateImageName(
                    repo_base: "docker.io/logdna",
                    , name: "build-images"
                    , variant_base: "rust"
                    , variant_version: "rust-${VARIANT_VERSION}"
                    , version: "${RUSTC_VERSION}"
                    , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}-amd64"
                    )
                def docker_arm64_image_name = generateImageName(
                    repo_base: "docker.io/logdna",
                    , name: "build-images"
                    , variant_base: "rust"
                    , variant_version: "rust-${VARIANT_VERSION}"
                    , version: "${RUSTC_VERSION}"
                    , image_suffix: "${CROSS_COMPILER_TARGET_ARCH}-arm64"
                    )
                // GCR image
                sh("docker manifest create ${gcr_image_name} --amend ${gcr_arm64_image_name} --amend ${gcr_amd64_image_name}")
                sh("docker push ${gcr_image_name}")
                // Dockerhub image
                sh("docker manifest create ${docker_image_name} --amend ${docker_arm64_image_name} --amend ${docker_amd64_image_name}")
                docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-username-password') {
                  sh("docker push ${docker_image_name}")
                }
              }
            }
          }
        }
      }
    }
  }
}

def generateImageName(Map config = [:]){
  String repo_base = "us.gcr.io/logdna-k8s"
  assert config.name : "Missing config.name"
  assert config.variant_base : "Missing config.variant_base"
  assert config.variant_version : "Missing config.variant_version"
  assert config.version : "Missing config.version"

  if (config.repo_base) {
    repo_base = config.repo_base
  }

  if (config.image_suffix) {
    return "${repo_base}/${config.name}:${config.variant_version}-1-${config.version}-${config.image_suffix}"
  } else {
    return "${repo_base}/${config.name}:${config.variant_version}-1-${config.version}"
  }
}

// Build and optionally push image
def buildImage(Map config = [:]) {
  assert config.name : "Missing config.name"
  assert config.variant_base : "Missing config.variant_base"
  assert config.variant_version : "Missing config.variant_version"
  assert config.version : "Missing config.version"

  // PR jobs have CHANGE_BRANCH set correctly
  // branch jobs have BRANCH_NAME set correctly
  // Neither are consistent, so we have to do this :[]
  def shouldPush =  ((env.CHANGE_BRANCH  == "main" || env.BRANCH_NAME == "main" ) || env.PUBLISH_IMAGE || config.push)

  def directory = "${config.name}/${config.variant_base}"

  List<String> buildArgs = [
    "--progress"
  , "plain"
  ]

  if (env.SCCACHE_BUCKET != null && env.SCCACHE_REGION != null) {
    buildArgs.push("--build-arg")
    buildArgs.push(["SCCACHE_BUCKET", env.SCCACHE_BUCKET].join("="))
    buildArgs.push("--build-arg")
    buildArgs.push(["SCCACHE_REGION", env.SCCACHE_REGION].join("="))
  } else {
    buildArgs.push("--build-arg")
    buildArgs.push("RUSTC_WRAPPER=")
    buildArgs.push("--build-arg")
    buildArgs.push("CC_WRAPPER=")
  }

  if (config.pull) {
    buildArgs.push("--pull")
  }

  if (config.base_name) {
    buildArgs.push("--build-arg")
    buildArgs.push(["BASE_IMAGE", config.base_name].join("="))
  }

  if (config.dockerfile) {
    buildArgs.push("-f")
    buildArgs.push([directory, config.dockerfile].join("/"))
  }

  if (config.cross_compiler_target_arch) {
    buildArgs.push("--build-arg")
    buildArgs.push(["CROSS_COMPILER_TARGET_ARCH", config.cross_compiler_target_arch].join("="))
  }

  if (config.platform) {
    buildArgs.push("--platform")
    buildArgs.push(config.platform)
  }

  buildArgs.push("--secret")
  buildArgs.push("id=aws,src=${env.WORKSPACE}/.aws_creds")

  buildArgs.push("--build-arg")
  buildArgs.push(["VERSION", config.version].join("="))

  buildArgs.push("--build-arg")
  buildArgs.push(["VARIANT_VERSION", config.variant_version].join("="))

  buildArgs.push(directory)

  def image = docker.build(config.image_name, buildArgs.join(' '))

  if (shouldPush) {
    image.push()
  }

  if (config.clean) {
    gcr.clean(image.id)
  }

  return image
}
