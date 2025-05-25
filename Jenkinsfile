pipeline {
    agent any

    environment {
        CSV_FILE = "${WORKSPACE}/merged_branches.csv"
        OUTPUT_FILE = "${WORKSPACE}/branch_details.txt"
        REPO_FILE = "${WORKSPACE}/repositories.txt"
        GITHUB_CREDENTIAL = credentials('githubCredentialsId')
        BRANCH_AGE_LIMIT = 30
    }

    stages {

        stage('Setup Workspace') {
            steps {
                script {
                    cleanWs()
                    writeFile file: env.CSV_FILE, text: "REPO_NAME,BRANCH_NAME,LAST_COMMIT_DATE,DAYS_OLD\n"
                }
            }
        }

        stage('Checkout Cleanup Repo & Validate Repo List') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: 'feature/cleanup']],
                        userRemoteConfigs: [[
                            url: "https://github.com/AmitaJoshi/BranchCleanup.git",
                            credentialsId: "${env.GITHUB_CREDENTIAL}"
                        ]]
                    ])
                    if (!fileExists(REPO_FILE)) {
                        error "File ${REPO_FILE} does not exist"
                    }
                }
            }
        }

        stage('Clone Repositories and List Branches') {
            steps {
                script {
                    def repoList = readFile(REPO_FILE).split('\n').findAll { it.trim() }
                    echo "Repo List: ${repoList}"

                    repoList.each { repoName ->
                        dir("${WORKSPACE}") {
                            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'githubCredentialsId',
                                usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
                                sh """
                                    git config --global http.timeout 900
                                    git clone https://github.com/AmitaJoshi/${repoName}.git
                                    cd ${repoName}
                                    git fetch --all
                                    branches=\$(git branch -r | grep "origin/" | sed 's/^ [* ]*//')
                                    echo "Repository: ${repoName}" >> "${OUTPUT_FILE}"
                                    echo "No of branches: \$(echo "\${branches}" | wc -l)" >> "${OUTPUT_FILE}"
                                    echo "Branches:" >> "${OUTPUT_FILE}"
                                    echo "--------" >> "${OUTPUT_FILE}"
                                    echo "\${branches}" >> "${OUTPUT_FILE}"
                                    echo "" >> "${OUTPUT_FILE}"
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Collect Old Merged Branches') {
            steps {
                script {
                    def repoList = readFile(REPO_FILE).split('\n').findAll { it.trim() }
                    def ageLimit = env.BRANCH_AGE_LIMIT.toInteger()

                    repoList.each { repoName ->
                        dir("${WORKSPACE}/${repoName}") {
                            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'githubCredentialsId',
                                usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {

                                sh """
                                    git config --global --add safe.directory '*'
                                    git fetch origin "+refs/heads/*:refs/remotes/origin/*"

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

                                    echo "Merged Branches (older than ${ageLimit} days):" >> "${OUTPUT_FILE}"
                                    echo "--------" >> "${OUTPUT_FILE}"

                                    for branch in \${merged_branches}; do
                                        last_commit_date_epoch=\$(git log -1 --format=%ct "\${branch}" 2>/dev/null || echo 0)
                                        if [ "\$last_commit_date_epoch" -eq 0 ]; then continue; fi

                                        last_commit_date=\$(date -d "@\${last_commit_date_epoch}" +"%Y-%m-%d %H:%M:%S")
                                        days_old=\$(( (current_date_epoch - last_commit_date_epoch) / 86400 ))

                                        if [ "\$days_old" -gt ${ageLimit} ]; then
                                            echo "${repoName},\${branch},\${last_commit_date},\${days_old}" >> "${CSV_FILE}"
                                        fi
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