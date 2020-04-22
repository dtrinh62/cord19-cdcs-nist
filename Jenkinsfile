pipeline {
    parameters {
        string(name: 'node_label', defaultValue: "master")
        string(name: 'data_url', defaultValue: "https://my-covid-data.org/dataset.tar.gz")
        string(name: 'access_token', defaultValue: "my_access_token")
        string(name: 'repository', defaultValue: "username/repository")
        string(name: 'tag', defaultValue: "release_tag")
        string(name: 'name', defaultValue: "release_name")
    }

    agent { node {label params.node_label} }
    stages {
        stage('Download') {
            steps {
                git branch: 'dev-automation',
                    credentialsId: 'jenkins-ssh-git',
                    url: 'git@github.com:pdessauw/cord19-cdcs-nist.git'

                dir ('build') {
                    sh """
                        # Create necessary directories
                        mkdir -p ./data ../dist/

                        # Download archive and untar
                        wget --no-check-certificate -q \
                            ${params.data_url} -O data.tar.gz
                    """
                }
            }
        }
        stage("Prepare data") {
            steps {
                dir ('build') {
                    sh """
                        # Unzip directory
                        tar -xvzf ./data.tar.gz --strip-components=1

                        # Unzip JSON files for build
                        unzip -q -o JSON.zip -d ./data

                        # Move zip to dist folder
                        cp *.zip -t ../dist/
                    """
                }
            }
        }
        stage('Install dependencies') {
            steps {
                dir ('python-interface') {
                    sh """
                        python3 -m pip install -r ../requirements.txt
                        python3 -m poetry install
                    """
                }
            }
        }
        stage('Build') {
            steps {
                dir ('python-interface') {
                    sh """
                        python3 -m poetry run task data-tf -o
                        python3 -m poetry build

                        cp ./dist/*.tar.gz ../dist/
                    """
                }
            }
        }
        stage('Release') {
            steps {
                dir ('dist') {
                    script {
                        def release_url = "https://api.github.com/repos/${params.repository}/releases"
                        def assets_url = "https://uploads.github.com/repos/${params.repository}/releases"

                        def release = sh(script: """
                            # Create the Github release
                            curl -XPOST -H "Authorization:token ${params.access_token}" \
                                --data '{
                                    "tag_name": "${params.tag}",
                                    "target_commitish": "master",
                                    "name": "${params.name}",
                                    "draft": false, "prerelease": true
                                }' $release_url
                        """, returnStdout: true)

                        def release_id = sh(script: """
                            echo "$release" | grep "id: " | head -n1 | \
                                cut -d':' -f2 | sed -r "s;[^0-9];;g"
                        """, returnStdout: true).trim()

                        def files = sh(
                            script: 'ls', returnStdout: true
                        ).split()

                        for (int i = 0; i < files.size(); i++) {
                            def filename = files[i]
                            sh """
                                curl -XPOST -H "Authorization:token ${params.access_token}" \
                                    -H "Content-Type:application/octet-stream" \
                                    --data-binary @$filename \
                                    $assets_url/$release_id/assets?name=$filename
                            """
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            echo "Cleaning up..."
            deleteDir()
        }
    }
}
