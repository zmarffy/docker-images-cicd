@Library('zekes-jenkins-tools') _

def HAS_DOCKER_LABEL = getConfigValue('HAS_DOCKER_LABEL', defaultValue: 'hasdocker')
def CHECKOUT_NODE = getConfigValue('CHECKOUT_NODE', defaultValue: 'built-in')
def TAGS_TO_BUILD = []
def ARCHS_TO_BUILD_FOR = []
def SOURCE_FOLDER = 'source'
def REPO_FOLDER = '_repo'
def IMAGES_FOLDER
def DOCKER_IMAGES_YAML_FILE = "$REPO_FOLDER/cicd/docker-images.yaml"
def HAS_DOCKER_IMAGES_YAML_FILE
def DOCKER_IMAGES_YAML_FILE_DATA
def COMPUTER_DATA = []
def USE_DOCKER_HUB
def INSECURE_REGISTRY
def NAME_ARCH_TAG_SHELL_STRING = '$IMAGE_NAME-$IMAGE_ARCH:$IMAGE_TAG'
def NAME_TAG_SHELL_STRING = '$IMAGE_NAME:$IMAGE_TAG'

pipeline {
    agent none
    options {
        buildDiscarder(
            logRotator(
                numToKeepStr: '100',
                artifactNumToKeepStr: '50'
            )
        )
    }
    parameters {
        string(name: 'GIT_LOCATION', description: 'location of project to build')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'branch of project to release from')
        string(name: 'GIT_CREDS_ID', description: 'creds ID to use to pull project from remote')
        booleanParam(name: 'PUSH', description: 'whether or not to push the image to a registry')
        booleanParam(name: 'SAVE', description: 'whether or not to save the Docker image to an artifact; note that these can be very big files')
        string(name: 'REGISTRY', description: 'registry to push to; leave blank for Docker Hub')
        string(name: 'REGISTRY_CREDS_ID', description: 'creds ID to use to push project to registry')
        booleanParam(name: 'CREATE_SINGLE_MANIFEST_FOR_ALL_ARCHS', defaultValue: true, description: 'create a manifest that points to images of all built architectures; can only be used if PUSH is true')
        booleanParam(name: 'CLEAN_UP_SINGLE_ARCH_REPOS', defaultValue: true, description: 'delete single-architecture repos; can only be used if PUSH is true, REGISTRY is blank (Docker Hub), and CREATE_SINGLE_MANIFEST_FOR_ALL_ARCHS is true')
        string(name: 'IMAGES_FOLDER', defaultValue: 'images', description: 'location where specifically versioned Docker images are in repo')
        string(name: 'IMAGE_NAME', description: 'name of image when built; leave blank to use git repo name')
        string(name: 'ARCHS', description: 'platforms to build images for, space delimited; leave blank for all')
        string(name: 'TAGS', description: 'tags of images to build, space delimited; leave blank for all')
    }
    stages {
        stage('Checkout') {
            agent {
                label CHECKOUT_NODE
            }
            steps {
                script {
                    checkout(
                        [
                            $class: 'GitSCM',
                            branches: [
                                [
                                    name: params.GIT_BRANCH
                                ]
                            ],
                            doGenerateSubmoduleConfigurations: false,
                            extensions: [
                                [
                                    $class: 'RelativeTargetDirectory',
                                    relativeTargetDir: REPO_FOLDER
                                ]
                            ],
                            submoduleCfg: [],
                            userRemoteConfigs: [
                                [
                                    credentialsId: params.GIT_CREDS_ID,
                                    url: params.GIT_LOCATION
                                ]
                            ]
                        ]
                    )
                    HAS_DOCKER_IMAGES_YAML_FILE = fileExists(DOCKER_IMAGES_YAML_FILE)
                    if (HAS_DOCKER_IMAGES_YAML_FILE) {
                        DOCKER_IMAGES_YAML_FILE_DATA = readYaml(file: DOCKER_IMAGES_YAML_FILE)
                    }
                    stash(
                        name: SOURCE_FOLDER,
                        includes: "$REPO_FOLDER/**"
                    )
                }
            }
        }
        stage('Set vars') {
            agent {
                label CHECKOUT_NODE
            }
            steps {
                script {

                    if (!params.PUSH && !params.SAVE) {
                        echo('As PUSH and SAVE are false, the created Docker images will not be pushed or saved anywhere')
                    }

                    def allTags
                    def allArchs
                    def imageName

                    IMAGES_FOLDER = "$REPO_FOLDER/$params.IMAGES_FOLDER"

                    if (!params.IMAGE_NAME) {
                        imageName = params.GIT_LOCATION.split('/')[-1]
                    }
                    else {
                        imageName = params.IMAGE_NAME
                    }
                    
                    USE_DOCKER_HUB = !params.REGISTRY
                    if (USE_DOCKER_HUB && !params.REGISTRY_CREDS_ID) {
                        error('Need to specify registry credentials ID when pushing to Docker Hub')
                    }
                    if (USE_DOCKER_HUB) {
                        env.REGISTRY = ''
                        withCredentials(
                            [
                                usernamePassword(
                                    credentialsId: params.REGISTRY_CREDS_ID,
                                    usernameVariable: 'REGISTRY_USERNAME',
                                    passwordVariable: '_unused'
                                )
                            ]
                        ) {
                            env.IMAGE_NAME = "$env.REGISTRY_USERNAME/$imageName"
                        }
                        INSECURE_REGISTRY = false
                    }
                    else {
                        INSECURE_REGISTRY = (!USE_DOCKER_HUB && !params.REGISTRY.startsWith('https'))
                        if (params.REGISTRY.endsWith('/')) {
                            env.REGISTRY = params.REGISTRY[0..-2]
                        }
                        else {
                            env.REGISTRY = params.REGISTRY
                        }
                        env.IMAGE_NAME = "${env.REGISTRY.split('://', 2)[-1]}/$imageName"
                    }

                    // Get tags to build
                    def possibleTags
                    if (HAS_DOCKER_IMAGES_YAML_FILE) {
                        possibleTags = DOCKER_IMAGES_YAML_FILE_DATA.collect { it['TAGS'] }.flatten()
                    }
                    else {
                        dir(IMAGES_FOLDER) {
                            possibleTags = findFiles().findAll { fileOrDirectory -> fileOrDirectory.name != '.git' && fileOrDirectory.directory }.collect { directory -> directory.name }.findAll { tagName -> findFiles(glob: "$tagName/Dockerfile") }
                        }
                    }
                    if (!possibleTags) {
                        error('No images to build')
                    }
                    TAGS_TO_BUILD = params.TAGS.split(' ').findAll { it != '' }
                    if (!TAGS_TO_BUILD) {
                        allTags = true
                        TAGS_TO_BUILD = possibleTags
                    }
                    else {
                        allTags = false
                        def invalidTags = TAGS_TO_BUILD.findAll { !(it in possibleTags) }
                        if (invalidTags) {
                            error("Invalid tags: $invalidTags")
                        }
                    }

                    // Get archs to build
                    def hasDockerNodeNames = nodesByLabel(
                        label: 'hasdocker',
                        offline: false
                    )
                    def hasDockerComputers = jenkins.model.Jenkins.get().computers.findAll { it.name in hasDockerNodeNames }
                    hasDockerComputers.each {
                        COMPUTER_DATA.add(
                            [
                                'name': it.name,
                                'arch': it.getSystemProperties().get('os.arch')
                            ]
                        )
                    }
                    def possibleArchs = COMPUTER_DATA.collect { it.arch }.unique()
                    if (!possibleArchs) {
                        // Impossible...
                        error('No archs to build for')
                    }
                    ARCHS_TO_BUILD_FOR = params.ARCHS.split(' ').findAll { it != '' }
                    if (!ARCHS_TO_BUILD_FOR) {
                        allArchs = true
                        ARCHS_TO_BUILD_FOR = possibleArchs
                    }
                    else {
                        allArchs = false
                        def invalidArchs = ARCHS_TO_BUILD_FOR.findAll { !(it in possibleArchs) }
                        if (invalidArchs) {
                            error("Invalid archs: $invalidArchs")
                        }
                    }

                    currentBuild.displayName = "$imageName ${allTags ? '(all tags)' : TAGS_TO_BUILD} ${allArchs ? '(all archs)' : ARCHS_TO_BUILD_FOR} -> ${env.REGISTRY ?: 'Docker Hub'}"
                }
            }
            post {
                always {
                    cleanWs()
                }
            }
        }
        stage('Build and push/save') {
            steps {
                script {
                    def parallelStages = [:]
                    ARCHS_TO_BUILD_FOR.each { arch ->
                        parallelStages[arch] = {
                            node(COMPUTER_DATA.find { it['arch'] == arch }.name) {
                                try {
                                    unstash(SOURCE_FOLDER)
                                    dir(IMAGES_FOLDER) {
                                        def buildPushSave = {
                                            for (tag in TAGS_TO_BUILD) {
                                                def envVars = [
                                                    'IMAGE_ARCH': arch,
                                                    'IMAGE_TAG': tag
                                                ]
                                                def buildScript = "docker build \"\$FOLDER\" -t $NAME_ARCH_TAG_SHELL_STRING -t \"\$IMAGE_NAME:\$IMAGE_TAG\""
                                                if (HAS_DOCKER_IMAGES_YAML_FILE) {
                                                    def yamlEntry = DOCKER_IMAGES_YAML_FILE_DATA.find { tag in it['TAGS'] }
                                                    yamlEntry['BUILD_ARGS'].eachWithIndex { buildArg, index ->
                                                        def k = "BUILD_ARG_$index"
                                                        envVars[k] = "$buildArg.key=$buildArg.value"
                                                        buildScript += " --build-arg \"\$$k\""
                                                    }
                                                    envVars['FOLDER'] = yamlEntry['FOLDER']
                                                }
                                                else {
                                                    envVars['FOLDER'] = tag
                                                }
                                                withEnvVars(envVars) {
                                                    sh(
                                                        script: buildScript,
                                                        label: 'Build Docker image'
                                                    )
                                                    if (params.PUSH) {
                                                        sh(
                                                            script: "docker push $NAME_ARCH_TAG_SHELL_STRING",
                                                            label: 'Push Docker image'
                                                        )
                                                    }
                                                    else if (params.SAVE) {
                                                        def tarFile = "$IMAGE_NAME-${arch}__${tag}.tar"
                                                        withEnvVars(
                                                            [
                                                                'TAR_FILE': tarFile
                                                            ]
                                                        ) {
                                                            sh(
                                                                script: "docker save $NAME_ARCH_TAG_SHELL_STRING -o \"\$TAR_FILE\"",
                                                                label: 'Save docker image'
                                                            )
                                                        }
                                                        archiveArtifacts(
                                                            artifacts: tarFile,
                                                            allowEmptyArchive: false,
                                                            fingerprint: true,
                                                        )
                                                    }
                                                }
                                            }
                                        }
                                        if (params.REGISTRY_CREDS_ID) {
                                            withDockerRegistry(
                                                credentialsId: params.REGISTRY_CREDS_ID,
                                                url: env.REGISTRY
                                            ) {
                                                buildPushSave()
                                            }
                                        }
                                        else {
                                            buildPushSave()
                                        }
                                    }
                                    for (tag in TAGS_TO_BUILD) {
                                        def envVars = [
                                            'IMAGE_TAG': tag
                                        ]
                                        withEnvVars(envVars) {
                                            sh(
                                                script: "docker rmi \"$NAME_TAG_SHELL_STRING\"",
                                                label: 'Remove temporary image tags'
                                            )
                                        }
                                    }
                                }
                                finally {
                                    cleanWs()
                                }
                            }
                        } 
                    }
                    // Do all arch builds at once!
                    parallel(parallelStages)
                }
            }
        }
        stage('Manifest') {
            agent {
                label HAS_DOCKER_LABEL
            }
            when {
                expression {
                    params.PUSH && params.CREATE_SINGLE_MANIFEST_FOR_ALL_ARCHS
                }
            }
            steps {
                script {
                    def manifest = {
                        for (tag in TAGS_TO_BUILD) {
                            def envVars = [
                                'IMAGE_TAG': tag
                            ]
                            def manifestScript = "docker manifest create --amend \"$NAME_TAG_SHELL_STRING\""
                            if (INSECURE_REGISTRY) {
                                manifestScript += ' --insecure'
                            }
                            ARCHS_TO_BUILD_FOR.eachWithIndex { arch, i ->
                                def e = "IMGNAME$i"
                                envVars[e] = "$env.IMAGE_NAME-$arch:$tag"
                                manifestScript += " \"\$$e\""
                            }
                            withEnvVars(envVars) {
                                sh(
                                    script: manifestScript,
                                    label: 'Update manifest'
                                )
                                sh(
                                    script: "docker manifest push --purge \"$NAME_TAG_SHELL_STRING\"",
                                    label: 'Push manifest'
                                )
                            }
                        }
                    }
                    if (params.REGISTRY_CREDS_ID) {
                        withDockerRegistry(
                            credentialsId: params.REGISTRY_CREDS_ID,
                            url: env.REGISTRY
                        ) {
                            manifest()
                        }
                    }
                    else {
                        manifest()
                    }
                    if (USE_DOCKER_HUB && params.CLEAN_UP_SINGLE_ARCH_REPOS) {
                        def dockerHubToken
                        withCredentials(
                            [
                                usernamePassword(
                                    credentialsId: params.REGISTRY_CREDS_ID,
                                    usernameVariable: 'REGISTRY_USERNAME',
                                    passwordVariable: 'REGISTRY_PASSWORD'
                                )
                            ]
                        ) {
                            dockerHubToken = readJSON(
                                text: performHttpRequest(
                                    'https://hub.docker.com/v2/users/login/',
                                    httpMode: 'POST',
                                    contentType: 'APPLICATION_JSON',
                                    requestBody: writeJSON(
                                        json: [
                                            'username': env.REGISTRY_USERNAME,
                                            'password': env.REGISTRY_PASSWORD
                                        ],
                                        returnText: true
                                    )
                                ).getContent()
                            )['token']
                        }
                        for (arch in ARCHS_TO_BUILD_FOR) {
                            echo("Deleting single-arch repo for arch $arch")
                            performHttpRequest(
                                "https://hub.docker.com/v2/repositories/$env.IMAGE_NAME-$arch",
                                httpMode: 'DELETE',
                                contentType: 'APPLICATION_JSON',
                                headers: [
                                    ['Authorization', "JWT $dockerHubToken"]
                                ]
                            )
                        }
                    }
                }
            }
            post {
                always {
                    cleanWs()
                }
            }
        }
    }
}
