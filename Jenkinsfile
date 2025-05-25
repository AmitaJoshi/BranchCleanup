pipeline {
    agent any
    /* parameters {
        choice(name: 'REPO_NAMES', choices: ['BranchCleanup'], description: 'select a repository')
    } */
    environment {
        CSV_FILE= "${WORKSPACE}/merged_branches.csv"
        GITHUB_API_URL = "https://bitbucket.org/your_workspace/${params.REPO_NAMES}"
        GITHUB_CREDENTIAL = credentials('githubCredentialsId')
        RESPONSE_FILE = "response.json"
        REPO_FILE = "repositories.txt"
        OUTPUT_FILE = "branch_details.txt"
    }
    stages {
        stage('SetUp') {
            steps {
                script {
                    cleanWs()
                    writeFile file: env.CSV_FILE, text: "REPO_NAME,BRANCH_NAME,LAST_COMMIT_DATE,DAYS_OLD\n"
                }
            }
        }
        stage('Branches Count'){
            steps{
                script {
                    dir("${env.WORKSPACE}"){
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: 'feature/cleanup']],
                            userRemoteConfigs: [[
                                url: "https://github.com/AmitaJoshi/BranchCleanup.git",
                                credentialsId: "${env.GITHUB_CREDENTIAL}"
                            ]]
                        ])
                        if (!fileExists(REPO_FILE)){
                            error "File ${REPO_FILE} does not exist"
                        }
                        def repoList = readFile(REPO_FILE).split('\n').findAll { it.trim() }
                        print "Repo List is :"+repoList
                        repoList.each { repoName ->
                           print "repo name ="+repoName
                           dir ("${env.WORKSPACE}"){
                                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:'githubCredentialsId',
                                usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
                                     sh """
                                        git config --global http.timeout 900
                                        git clone 'https://github.com/AmitaJoshi/${repoName}'
                                        cd ${repoName} 
                                        git fetch --all
                                        branches=\$(git branch -r | grep "origin/*" | sed 's/^ [* ]*//')
                                        branch_count=\$(echo "\${branches}" | wc -l)
                                        echo "Repository: ${repoName}" >> "${env.WORKSPACE}"/"${OUTPUT_FILE}"
                                        echo "No of branches are : \${branch_count}" >> "${env.WORKSPACE}"/"${OUTPUT_FILE}"
                                        echo "Branches:" >> "${env.WORKSPACE}"/"${OUTPUT_FILE}"
                                        echo "--------" >> "${env.WORKSPACE}"/"${OUTPUT_FILE}"
                                        echo "\${branches}" >> "${env.WORKSPACE}"/"${OUTPUT_FILE}"
                                        echo "\n"
                                        echo "##################################################"
                                        echo "\n"
                                    """
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('get Old Merged Branches'){ 
            steps {
                script {
                    if(!fileExists(REPO_FILE)){
                        error "File ${REPO_FILE} does not exist"
                    }
                    env.BRANCH_AGE_LIMIT = 30
                    def repoList = readFile(REPO_FILE).split('\n').findAll { it.trim() }
                    print "Repo List is :"+repoList
                    sh (script : "echo 'repository Name, branch name, last commit date, days old' > ${env.CSV_FILE}")
                    repoList.each { repoName ->
                        print "repo name ="+repoName
                        dir("${env.WORKSPACE}"){
                            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:'githubCredentialsId',
                            usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']])
                            {
                                sh """
                                    current_date=\$(date +%s)
                                    ls -lart 
                                    cat \$CSV_FILE
                                    echo "\$current_date"
                                    echo "\$repoName"
                                    cd "\$WORKSPACE/\$repoName"
                                    
                                    git fetch --all
                                    git branch -r | grep "origin/feature" | sed 's/^ [* ]//' > branches.txt
                                    echo "Text file created" 
                                    cat branches.txt 
                                    pwd

                                    while read -r branch; do 
                                        branch_name=\$(echo "\$branch" | sed 's|origin/||')
                                        last_commit_date=\$(git log -1 --format="%ct" "\$branch")
                                        branch_age_days=\$(( (current_date - last_commit_date) / (60*60*24) ))
                                        formatted_last_commit_date=\$(date -d "@\$last_commit_date" +"%d-%B-%Y")

                                        if [ "\$branch_age_days" -gt 1 ]; then
                                            echo "\$repoName, \$branch_name, \$formatted_last_commit_date, \$branch_age_days"
                                        fi
                                    done < branches.txt

                                    release_branches=\$(git branch -r | grep "origin/release" | sed 's/^ [* ]//')
                                """
                            }
                        }
                    }
                    dir("${env.WORKSPACE}"){
                        //def repoList = readFile(REPO_FILE).split('\n').findAll { it.trim() }
                        print "Repo List is :"+repoList
                        repoList.each { repoName ->
                            print "repo name ="+repoName
                            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:'githubCredentialsId',
                            usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]){
                            sh """
                                cd "${repoName}"
                                git config --global --add safe.directory '*'
                                git fetch origin "+refs/heads/*:refs/remotes/origin/*"
                                git ls-remote --heads origin
                                merged_branches=\$( \
                                { \
                                    git for-each-ref --format='%(refname:short)' refs/remotes/origin/release* 2>/dev/null | while read release_branch; do \
                                    git branch -r --merged "\$release_branch"; \
                                    done; \
                                    git branch -r --merged origin/main 2>/dev/null || true; \
                                    git branch -r --merged origin/master 2>/dev/null || true; \
                                } | sort -u | grep -vE 'origin/(master|main|develop|release|staging)' )

                                echo "Merged Branches:" >> "\${WORKSPACE}/\${OUTPUT_FILE}"
                                echo "--------" >> "\${WORKSPACE}/\${OUTPUT_FILE}"
                                for branch in \${merged_branches}; do
                                    echo "repo name inside for loop = \${repoName}"
                                    last_commit_date=\$(git log -1 --format=%cd --date=iso "\${branch}")
                                    last_commit_date_epoch=\$(date -d "\${last_commit_date}" +%s)
                                    current_date_epoch=\$(date +%s)
                                    days_old=\$(( (\${current_date_epoch} - \${last_commit_date_epoch}) / 86400 ))
                                    echo "\${repoName},\${branch},\${last_commit_date},\${days_old}" >> "\${CSV_FILE}"
                                done
                            """
                            }        
                        }
                    }
                }
            }
        }
    }
    post {
    always {
        archiveArtifacts artifacts: 'merged_branches.csv, branch_details.txt', onlyIfSuccessful: false
    }
}
}
