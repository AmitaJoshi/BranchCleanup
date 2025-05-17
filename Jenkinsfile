pipeline {
    agent any
    parameters {
        choice(name: 'REPO_NAMES', choices: ['BranchCleanup'], description: 'select a repository')
    }
    environment {
        CSV_FILE= "${WORKSPACE}/merged_branches.csv"
        GITHUB_API_URL = "https://bitbucket.org/your_workspace/${params.REPO_NAMES}"
        GITHUB_CREDENTIAL = credentials('Icg-git-basicauth')
        RESPONSE_FILE = "response.json"
        REPO_FILE = "repositories.txt"
        OUTPUT_FILE = "branch_details.txt"
    }
    stages {
        stage('SetUp') {
            steps {
                script {
                    deleteDir()
                    writeFile file: env.CSV_FILE, text: "REPO_NAME,BRANCH_NAME,LAST_COMMIT_DATE,DAYS_OLD\n"
                }
            }
        }
        stage('Branches Count') {
            steps {
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
                    }
                }
            }
        }
    }
}
