def label = "jenkins-agent"


// 执行Helm的方法
def helmDeploy(Map args) {
    if(args.init){
        println "Helm 初始化"
        sh "helm repo add myrepo ${args.url}"
    } else if (args.dry_run) {
        println "尝试 Helm 部署，验证是否能正常部署"
        sh "helm upgrade --install ${args.name} --namespace ${args.namespace} ${args.values} --set ${args.image},${args.tag} myrepo/${args.template} --dry-run --debug"
    } else {
        println "正式 Helm 部署"
        sh "helm upgrade --install ${args.name} --namespace ${args.namespace} ${args.values} --set ${args.image},${args.tag} myrepo/${args.template}"
    }
}

podTemplate(label: label,cloud: 'kubernetes' ){
    node (label) {
        
        echo "测试 kubernetes 中 jenkins slave 代理！~"
        
        stage('Git阶段'){
            echo "Git 阶段"
            git branch: "main" ,changelog: true , url: "https://gitee.com/angleslmh/springboot-helloworld.git"
        }
        stage('Maven阶段'){
            container('maven') {
                //这里引用上面设置的全局的 settings.xml 文件，根据其ID将其引入并创建该文件
                configFileProvider([configFile(fileId: "global-maven-settings", targetLocation: "settings.xml")]){
                    sh "mvn clean install -Dmaven.test.skip=true --settings settings.xml"
                }
            }
        }
        stage('Docker阶段'){
            echo "Docker 阶段"
            container('docker') {
                // 读取pom参数
                echo "读取 pom.xml 参数"
                pom = readMavenPom file: './pom.xml'
                // 设置镜像仓库地址
                hub = "docker.io"
                // 设置仓库项目名
                project_name = "maxfjxm"
                echo "编译 Docker 镜像"
                docker.withRegistry("https://index.docker.io/v1/", "docker-hub-credential") {
                    echo "构建镜像"
                    // 设置推送到hub仓库的maxfjxm项目下，并用pom里面设置的项目名与版本号打标签
                    def customImage = docker.build("${hub}/${project_name}/${pom.artifactId}:${pom.version}")
                    echo "推送镜像"
                    customImage.push()
                    echo "删除镜像"
                    sh "docker rmi ${hub}/${project_name}/${pom.artifactId}:${pom.version}" 
                }
            }
        }
        stage('Helm阶段'){
            container('helm-kubectl') {
                withKubeConfig([credentialsId: "b35480f1-d15b-49ac-aba2-29cceee9b37c",serverUrl: "https://192.168.8.132:6443"]) {
                    // 设置参数
                    image = "image.repository=${hub}/${project_name}/${pom.artifactId}"
                    tag = "image.tag=${pom.version}"
                    template = "spring-boot"
                    repo_url = "https://maikoulin.github.io/helm-chart"
                    app_name = "${pom.artifactId}"
                    // 检测是否存在yaml文件
                    def values = ""
                    if (fileExists('values.yaml')) {
                        values = "-f values.yaml"
                    }
                    // 执行 Helm 方法
                    echo "Helm 初始化"
                    helmDeploy(init: true ,url: "${repo_url}");
                    echo "Helm 执行部署测试"
                    helmDeploy(init: false ,dry_run: true ,name: "${app_name}" ,namespace: "jenkins" ,image: "${image}" ,tag: "${tag}" , values: "${values}" ,template: "${template}")
                    echo "Helm 执行正式部署"
                    helmDeploy(init: false ,dry_run: false ,name: "${app_name}" ,namespace: "jenkins",image: "${image}" ,tag: "${tag}" , values: "${values}" ,template: "${template}")
                }
            }
        }
    }
}
