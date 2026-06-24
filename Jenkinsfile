stage('Install Node Dependencies') {
    steps {
        sh 'npm ci'
    }
}

stage('Prepare Android') {
    steps {
        sh 'chmod +x android/gradlew'
        sh 'echo "sdk.dir=/opt/android-sdk" > android/local.properties'
    }
}

stage('Install Fastlane Dependencies') {
    steps {
        dir('android') {
            sh 'bundle config set path vendor/bundle'
            sh 'bundle install'
        }
    }
}

stage('Build Android Debug') {
    steps {
        dir('android') {
            sh 'bundle exec fastlane build_debug'
        }
    }
}