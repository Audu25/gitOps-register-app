pipeline {
  agent { label "Jenkins-Agent" }

  environment {
    APP_NAME = "register-app-pipeline"     // container name in k8s
    DOCKERHUB_NAMESPACE = "audu333"        // your Docker Hub user/org
    // Optional: let IMAGE_TAG be passed from upstream; else derive from build #
    // (If you already export IMAGE_TAG earlier, this fallback is ignored)
  }

  stages {
    stage("Cleanup Workspace") {
      steps { cleanWs() }
    }

    stage("Checkout from SCM") {
      steps {
        git branch: 'main', credentialsId: 'github',
            url: 'https://github.com/Audu25/gitOps-register-app.git'
      }
    }

    stage("Update the Deployment Tags") {
      steps {
        sh '''
          set -e
          # Derive tag if not set already
          IMAGE_TAG="${IMAGE_TAG:-1.0.0-${BUILD_NUMBER:-local}}"
          JD_TAGGED_IMAGE_NAME="${DOCKERHUB_NAMESPACE}/${APP_NAME}:${IMAGE_TAG}"
          echo "Updating image to: ${JD_TAGGED_IMAGE_NAME}"

          echo "---- BEFORE ----"
          grep -n "image:" deployment.yaml || true

          # Replace the whole image line, robustly
          sed -i -E "s#^(\\s*image:\\s*).+#\\1${JD_TAGGED_IMAGE_NAME}#" deployment.yaml

          echo "---- AFTER ----"
          grep -n "image:" deployment.yaml || true
        '''
      }
    }

    stage("Push the changed deployment file to Git") {
      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
          sh '''
            set -e
            git config --global user.name  "Audu25"
            git config --global user.email "auduabednego@yahoo.com"
            # (Optional) avoid "dubious ownership" if running as root
            git config --global --add safe.directory "$PWD"

            # Only commit if there are changes
            if git diff --quiet --exit-code -- deployment.yaml; then
              echo "No changes to commit."
              exit 0
            fi

            git add deployment.yaml
            git commit -m "chore(ci): update image to ${DOCKERHUB_NAMESPACE}/${APP_NAME}:${IMAGE_TAG}"

            # Make sure we can push (fetch in case main moved)
            git pull --rebase origin main || true

            # Push with creds from Jenkins binding (avoids 403)
            git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Audu25/gitOps-register-app.git
            git push origin HEAD:main
          '''
        }
      }
    }
  }
}
