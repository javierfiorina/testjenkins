node('master'){
    stage('stage1'){
    def commit = sh (returnStdout: true, script: '''echo hi
    echo bye | grep -o "e"
    date
    echo lol''').split()


    echo "${commit[-1]} "

    }
}

