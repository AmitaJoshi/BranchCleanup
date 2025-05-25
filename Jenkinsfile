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
            cd "${repoName}"
            git config --global --add safe.directory '*'
            git fetch origin "+refs/heads/*:refs/remotes/origin/*"
            git ls-remote --heads origin

            current_date_epoch=\$(date +%s)

            merged_branches=\$(
                {
                    git for-each-ref --format='%(refname:short)' refs/remotes/origin/release* 2>/dev/null | while read release_branch; do
                        git branch -r --merged "\$release_branch"
                    done
                    git branch -r --merged origin/main 2>/dev/null || true
                    git branch -r --merged origin/master 2>/dev/null || true
                } | sort -u | grep -vE 'origin/(master|main|develop|release|staging)'
            )

            echo "Merged Branches:" >> "\${WORKSPACE}/\${OUTPUT_FILE}"
            echo "--------" >> "\${WORKSPACE}/\${OUTPUT_FILE}"

            for branch in \${merged_branches}; do
                last_commit_date_epoch=\$(git log -1 --format=%ct "\${branch}" 2>/dev/null || echo 0)
                if [ "\${last_commit_date_epoch}" -eq 0 ]; then
                    continue
                fi
                last_commit_date=\$(date -d "@\${last_commit_date_epoch}" +"%Y-%m-%d %H:%M:%S")
                days_old=\$(( (current_date_epoch - last_commit_date_epoch) / 86400 ))

                if [ "\${days_old}" -gt ${BRANCH_AGE_LIMIT} ]; then
                    echo "${repoName},\${branch},\${last_commit_date},\${days_old}" >> "\${CSV_FILE}"
                fi
            done
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
                                            last_commit_date_epoch=\$(git log -1 --format=%ct "\${branch}" 2>/dev/null)
                                            last_commit_date=\$(date -d "@\${last_commit_date_epoch}" +"%Y-%m-%d %H:%M:%S")
                                            current_date_epoch=\$(date +%s)
                                            days_old=\$(( (current_date_epoch - last_commit_date_epoch) / 86400 ))

                                            echo "${repoName},\${branch},\${last_commit_date},\${days_old}" >> "\${CSV_FILE}"
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
