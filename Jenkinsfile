pipeline {
  agent {
    node {
      label 'master'
    }
  }
  stages {
    stage('Repo Sync') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''repo init -u https://github.com/Halium/android -b halium-7.1 --depth=1 && \\
(repo sync -c -j35 --force-sync) || (
repo forall -vc "git reset --hard" && \\
repo sync -c -j35 --force-sync) || (
rm -rf * && \\
repo sync -l && \\
repo sync -c -j35 --force-sync)'''
        }
      }
    }
    stage('Get Harpia sources') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''wget -O halium/devices/manifests/motorola_harpia.xml https://github.com/IkerST/manifest-halium/raw/master/motorola_harpia-hal.xml &&  \\
JOBS=35 ./halium/devices/setup harpia --force-sync'''
        }
      }
    }
/* Don't clean this time
    stage('Clean Enviroment') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''source build/envsetup.sh && \\
breakfast harpia && \\
mka clean'''
        }
      }
    }*/
    stage('Build Mkbootimg') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''source build/envsetup.sh && \\
breakfast harpia && \\
mka mkbootimg'''
        }
      }
    }
    stage('Build hybris-boot') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''source build/envsetup.sh && \\
breakfast harpia && \\
mka hybris-boot'''
        }
      }
    }
    stage('Build halium-boot') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''source build/envsetup.sh && \\
breakfast harpia && \\
mka halium-boot'''
        }
      }
    }
    stage('Build systemimage') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''source build/envsetup.sh && \\
breakfast harpia && \\
mka systemimage'''
        }
      }
    }
    stage('Archive Artifacts') {
      steps {
        ws('workspace/Halium-Build-Common') {
          archiveArtifacts(artifacts: 'out/target/product/harpia/*.img', excludes: 'android-ramdisk.img, dt.img, ramdisk-recovery.img, recovery.img')
        }
      }
    }
    stage('Upload Release') {
      steps {
        dir(path: 'release') {
          unarchive mapping: ['out/target/product/harpia/*.img' : '.']
          sh '''GIT_TAG=$(date +%d-%m-%y-%H-%M-%S) && \\
          pwd && eval $(ssh-agent -s) && \\
          ssh-add ~/.ssh/git && \\
          git clone git@github.com:HaliumForMSM8916/jenkins_build.git jenkins_build && \\
          cd jenkins_build && \\
          git checkout harpia-hal && \\
          git tag $GIT_TAG && git push --tags && \\
          cd .. && rm -rf jenkins_build && \\
          github-release release \\
    --security-token $(cat ~/.github_api_key) \\
    --user HaliumForMSM8916 \\
    --repo jenkins_build \\
    --tag $GIT_TAG \\
    --name "Halium Harpia $GIT_TAG" \\
    --description "This is a test build for the harpia" \\
    --pre-release && \\
          for a in $(cd out/target/product/harpia/ && ls -1 *.img); do github-release upload \\
    --security-token $(cat ~/.github_api_key) \\
    --user HaliumForMSM8916 \\
    --repo jenkins_build \\
    --tag $GIT_TAG \\
    --name $a \\
    --file out/target/product/harpia/$a ; done'''
          deleteDir()
        }
      }
    }
  }
}
