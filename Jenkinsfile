pipeline{
    agent any
    environment { // Step 2
        DB_RELEASE_URL = credentials('DB_RELEASE_URL')
        DB_LICENSE_KEY = credentials('DB_LICENSE_KEY')
        DB_LICENSE_LIC = credentials('DB_LICENSE_LIC')
        DB_SSH_KEY = credentials('DB_SSH_KEY')
    }

    stages {
        stage('sanity checks') {
            steps {
                cleanWs() // clean jenkins so it re-clones
                checkout scm // re-clone/re-download
                sh 'git branch'
                sh 'git status'
            }
        }

        stage('Build project') {
            steps { // Step 1
              sh 'mvn --batch-mode --no-transfer-progress clean install -DskipTests'
              // batch-mode = suppress upload messages to avoid polluting the console log
              // no-transfer-progress = remove some output; doanloading messages etc
              // -DskipTests = skipping tests to save time (tests assumed to compile and pass)
            }
        }

        stage('Activate and run dcover') {
            steps { // Step 3
                 sh '''
                      echo "Get and unzip dcover jars into directory dcover, store dcover script location for later use"
                      mkdir --parents dcover
                      wget "$DB_RELEASE_URL" --output-document dcover/dcover.zip --quiet
                      unzip -o dcover/dcover.zip -d dcover
                      DCOVER_SCRIPT_LOCATION="dcover/dcover"

                      echo "Activate dcover online"
                      "$DCOVER_SCRIPT_LOCATION" activate "$DB_LICENSE_KEY"

                      echo "Running dcover with create on create, you may need to update this to reflect your project"
                      "$DCOVER_SCRIPT_LOCATION" create --batch
                  '''

                 /*
                 sh ''' OFFLINE
                    echo "Activate dcover offline"
                    "$DCOVER_SCRIPT_LOCATION" license || true
                    cp "$DB_LICENSE_LIC" ${HOME}/.diffblue/offline/
                    "$DCOVER_SCRIPT_LOCATION" activate --offline "$DB_LICENSE_KEY"
                 '''
                 */
            }
        }

        stage('Commit new tests to git') {
            steps{
                sh '''
                    echo "Remote host set up"
                    eval "$(ssh-agent -s)"
                    ssh-add $DB_SSH_KEY
                    git config user.name db-ci-bot
                    git config user.email db-ci-bot-jane@diffblue.com

                   if [ -n "$(git status --short **/*DiffblueTest.java)" ]; then
                       git add **/*DiffblueTest.java
                       git commit --message "Update Unit Tests for $(git rev-parse --short HEAD)"
                       git push --set-upstream origin
                     else
                       echo "Nothing to commit"
                     fi
                '''
            }
        }
    }
}
