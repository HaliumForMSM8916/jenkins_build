pipeline {
  agent {
    node {
      label 'master'
    }
  }
  stages {
    stage('Notify Start') {
      steps {
        withCredentials([string(credentialsId: 'TG_BOT_API', variable: 'TG_BOT_API')]) {
          sh '''curl "https://api.telegram.org/bot"$TG_BOT_API"/sendMessage" -d "{ \\"chat_id\\":\\"-1001405063046\\", \\"text\\":\\"UT Build Started at: [jenkins](${BUILD_URL})\\", \\"parse_mode\\":\\"markdown\\"}" -H "Content-Type: application/json" -s > /dev/null'''
        }
      }
    }
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
          sh '''wget -O halium/devices/manifests/motorola_harpia.xml https://github.com/HaliumForMSM8916/manifest-halium/raw/master/motorola_harpia-ut.xml &&  \\
JOBS=35 ./halium/devices/setup harpia --force-sync'''
        }
      }
    }
    stage('Experimental UT Patches') {
      steps {
        ws('workspace/Halium-Build-Common') {
          sh '''(cd halium/libhybris/ && git remote add ubports https://github.com/ubports/libhybris.git && git fetch ubports && git checkout ubports/hybris-compat-layer-fixes) && \\
          (cd halium/ && git clone https://github.com/ubports/platform-api.git && cd platform-api && git checkout xenial) && \\
          (cd build && curl -L https://github.com/HaliumForMSM8916/jenkins_build/raw/harpia-ut-experimental/0001-UT-patch.patch | git am -  )'''
        }
      }
    }
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
    stage('Generate udev rules') {
      steps{
        ws('workspace/Halium-Build-Common') {
          sh '''cat out/target/product/harpia/root/ueventd*.rc | grep ^/dev | sed -e 's/^\\/dev\\///' | awk '{printf "ACTION==\\"add\\", KERNEL==\\"%s\\", OWNER=\\"%s\\", GROUP=\\"%s\\", MODE=\\"%s\\"\\n",$1,$3,$4,$2}' | sed -e 's/\\r//' > out/target/product/harpia/70-harpia.rules'''
        }
      }
    }
    stage('Archive Artifacts') {
      steps {
        ws('workspace/Halium-Build-Common') {
          archiveArtifacts(artifacts: 'out/target/product/harpia/*.img, out/target/product/harpia/70-harpia.rules', excludes: 'android-ramdisk.img, dt.img, ramdisk-recovery.img, recovery.img')
          cleanWs()
        }
      }
    }
    stage('Upload Release') {
      steps {
        dir(path: 'release') {
          unarchive mapping: ['out/target/product/harpia/*' : '.']
          withCredentials([
           string(credentialsId: 'TG_BOT_API', variable: 'TG_BOT_API'),
           string(credentialsId: 'GITHUB_API', variable: 'GIT_API_KEY')
          ]) {
            sh '''GIT_TAG=$(date +%d-%m-%y-%H-%M-%S) && \\
            pwd && eval $(ssh-agent -s) && \\
            ssh-add ~/.ssh/git && \\
            git clone git@github.com:HaliumForMSM8916/jenkins_build.git jenkins_build && \\
            cd jenkins_build && \\
            git checkout harpia-hal && \\
            git tag $GIT_TAG && git push --tags && \\
            cd .. && rm -rf jenkins_build && \\
            github-release release \\
      --security-token $GIT_API_KEY \\
      --user HaliumForMSM8916 \\
      --repo jenkins_build \\
      --tag $GIT_TAG \\
      --name "UT Experimental Harpia $GIT_TAG" \\
      --description "This is a test build for the harpia, when installing from zip the root/phablet password is 123456789" \\
      --pre-release && \\
            for a in $(cd out/target/product/harpia/ && ls -1 *); do github-release upload \\
      --security-token $GIT_API_KEY \\
      --user HaliumForMSM8916 \\
      --repo jenkins_build \\
      --tag $GIT_TAG \\
      --name $a \\
      --file out/target/product/harpia/$a ; \\
      md5sum out/target/product/harpia/$a >> sums.md5sum ; done
      git clone git@github.com:HaliumForMSM8916/halium-zip.git halium-zip && \\
      cd halium-zip && \\
      wget -q --show-progress https://ci.ubports.com/job/xenial-rootfs-armhf/lastSuccessfulBuild/artifact/out/ubports-touch.rootfs-xenial-armhf.tar.gz -O rootfs.tar.gz && \\
      ./halium-install -p ut -n rootfs.tar.gz  ../out/target/product/harpia/system.img ../out/target/product/harpia/hybris-boot.img harpia && \\
      ZIP=$(ls -1 *.zip)
      github-release upload \\
      --security-token $GIT_API_KEY \\
      --user HaliumForMSM8916 \\
      --repo jenkins_build \\
      --tag $GIT_TAG \\
      --name $ZIP \\
      --file $ZIP ; \\
      md5sum $ZIP >> ../sums.md5sum && \\
      cd ../ && \\
      sudo rm -rf halium-zip && \\
      github-release upload \\
      --security-token $GIT_API_KEY \\
      --user HaliumForMSM8916 \\
      --repo jenkins_build \\
      --tag $GIT_TAG \\
      --name sums.md5sum \\
      --file sums.md5sum ; \\
      RELEASE_URL=$(github-release info -u HaliumForMSM8916 -r jenkins_build -t $GIT_TAG -j | jq -r '.Releases[]|"\\(.html_url)"'); \\
      RELEASE_NAME=$(github-release info -u HaliumForMSM8916 -r jenkins_build -t $GIT_TAG -j | jq -r '.Releases[]|"\\(.name)"'); \\
      curl "https://api.telegram.org/bot${TG_BOT_API}/sendMessage" -d "{ \\"chat_id\\":\\"-1001405063046\\", \\"text\\":\\"New Release ${RELEASE_NAME}: [github](${RELEASE_URL})\\", \\"parse_mode\\":\\"markdown\\"}" -H "Content-Type: application/json" -s > /dev/null
      '''
          }
          deleteDir()
        }
      }
    }
  }
}
