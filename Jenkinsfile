pipeline {
  agent none
  environment {
    CACHE_DIR = "testsuite/cache/${env.CHANGE_TARGET ?: env.GIT_BRANCH}"
  }
  options {
    newContainerPerStage()
    disableConcurrentBuilds()
  }
  parameters {
    booleanParam(name: 'MSVC64', defaultValue: false, description: 'Build with MSVC64 (often hangs)')
    booleanParam(name: 'MINGW32', defaultValue: true, description: 'Build with MINGW32 (does not link boost)')
    string(name: 'RUNTESTS_FLAG', defaultValue: '', description: 'runtests.pl flag')
  }
  stages {
    stage('build') {
      parallel {
        stage('linux64') {
          agent {
            dockerfile {
              additionalBuildArgs '--pull'
              dir '.CI/cache'
              /* The cache Dockerfile makes /cache/runtest, etc world writable
               * This is necessary because we run the docker image as a user and need to
               * be able to have a global caching of the omlibrary parts and the runtest database.
               * Note that the database is stored in a volume on a per-node basis, so the first time
               * the tests run on a particular node, they might execute slightly slower
               */
              label 'linux'
              args "--mount type=volume,source=runtest-omsimulator-cache-linux64,target=/cache/runtest --tmpfs /tmp"
            }
          }
          environment {
            RUNTESTDB = "/cache/runtest/"
            NPROC = "${numPhysicalCPU}"
          }
          steps {
            buildOMS()
            sh '''
            # No so-files should ever exist in a bin/ folder
            ! ls install/linux/bin/*.so 1> /dev/null 2>&1
            '''

            partest()

            junit 'testsuite/partest/result.xml'

            // Temporary debugging
            // archiveArtifacts artifacts: 'testsuite/**/*.log', fingerprint: true

            sh 'make doc doc-html doc-doxygen'
            sh '(cd install/linux/doc && zip -r "../../../OMSimulator-doc-`git describe`.zip" *)'
            archiveArtifacts artifacts: 'OMSimulator-doc*.zip', fingerprint: true
            stash name: 'docs', includes: "install/linux/doc/**"
          }
        }
        stage('linux64-asan') {
          agent {
            node {
              label 'linux'
            }
          }
          environment {
            RUNTESTDB = "/cache/runtest/"
            ASAN = "ON"
          }
          steps {
            script {
              image = docker.build("build-deps-cache", "--pull .CI/cache")
              image.inside("-e RUNTESTDB=${env.RUNTESTDB} -e ASAN=${env.ASAN} --tmpfs /tmp") {
                // Keep ASAN jobs separate since you might get HEARTBEAT_CHECK_INTERVAL problems
                buildOMS()
              }
              image.inside("--mount type=volume,source=runtest-omsimulator-cache-linux64-asan,target=/cache/runtest " +
                     "--cap-add SYS_PTRACE --privileged " + // Needed for ASAN
                     "--oom-kill-disable -m 1024m --memory-swap 1024m " +
                     "-e RUNTESTDB=${env.RUNTESTDB} -e ASAN=${env.ASAN} --tmpfs /tmp")
              {
                timeout(20) { retry(2) {
                  partest()
                } }
              }
            }
            junit 'testsuite/partest/result.xml'
          }
        }
        stage('centos6') {
          agent {
            dockerfile {
              additionalBuildArgs '--pull'
              dir '.CI/centos6'
              label 'linux'
              args '--tmpfs /tmp'
            }
          }
          environment {
            SHELLSTART = """
            export CC="/opt/rh/devtoolset-7/root/usr/bin/gcc"
            export CXX="/opt/rh/devtoolset-7/root/usr/bin/g++"
            export CXXFLAGS="-std=c++17"
            """
            OMSFLAGS = "CERES=OFF OMSYSIDENT=OFF OMTLM=OFF"
          }
          steps {
            buildOMS()
            sh '''
            # No so-files should ever exist in a bin/ folder
            ! ls install/linux/bin/*.so 1> /dev/null 2>&1
            (cd install/linux && tar czf "../../OMSimulator-linux-amd64-`git describe`.tar.gz" *)
            '''
            archiveArtifacts "OMSimulator-linux-amd64-*.tar.gz"
          }
        }
        stage('alpine') {
          agent {
            dockerfile {
              additionalBuildArgs '--pull'
              dir '.CI/alpine'
              label 'linux'
            }
          }
          environment {
            OMSFLAGS = "CERES=OFF OMSYSIDENT=OFF OMTLM=OFF"
          }
          steps {
            buildOMS()
          }
        }
        stage('linux32') {
          agent {
            dockerfile {
              additionalBuildArgs '--pull'
              dir '.CI/cache-32'
              /* The cache Dockerfile makes /cache/runtest, etc world writable
               * This is necessary because we run the docker image as a user and need to
               * be able to have a global caching of the omlibrary parts and the runtest database.
               * Note that the database is stored in a volume on a per-node basis, so the first time
               * the tests run on a particular node, they might execute slightly slower
               */
              label 'linux'
              args "--mount type=volume,source=runtest-omsimulator-cache-linux32,target=/cache/runtest "+
                   '--tmpfs /tmp'
            }
          }
          environment {
            RUNTESTDB = "/cache/runtest/"
          }
          steps {
            buildOMS()
            sh '''
            # No so-files should ever exist in a bin/ folder
            ! ls install/linux/bin/*.so 1> /dev/null 2>&1
            (cd install/linux && tar czf "../../OMSimulator-linux-i386-`git describe`.tar.gz" *)
            '''

            archiveArtifacts "OMSimulator-linux-i386-*.tar.gz"

            partest()

            // Disable until working
            // junit 'testsuite/partest/result.xml'
          }
        }
        stage('linux-arm32') {
          stages {
            stage('cross-compile') {
              agent {
                docker {
                  image 'docker.openmodelica.org/armcross-omsimulator:v2.0'
                  label 'linux'
                  alwaysPull true
                  args '--tmpfs /tmp'
                }
              }
              environment {
                CROSS_TRIPLE = "arm-linux-gnueabihf"
                CC = "${env.CROSS_TRIPLE}-gcc"
                CXX = "${env.CROSS_TRIPLE}-g++"
                AR = "${env.CROSS_TRIPLE}-ar"
                RANLIB = "${env.CROSS_TRIPLE}-ranlib"
                FMIL_FLAGS = '-DFMILIB_FMI_PLATFORM=arm-linux-gnueabihf'
                detected_OS = 'Linux'
                VERBOSE = '1'
                OMSFLAGS = "OMTLM=OFF"
              }
              steps {
                buildOMS()
                sh '''
                # No so-files should ever exist in a bin/ folder
                ! ls install/linux/bin/*.so 1> /dev/null 2>&1
                (cd install/linux && tar czf "../../OMSimulator-linux-arm32-`git describe`.tar.gz" *)
                '''

                archiveArtifacts "OMSimulator-linux-arm32-*.tar.gz"
                stash name: 'arm32-install', includes: "install/linux/**"
              }
            }

            stage('test') {
              /* when {
                beforeAgent true
                expression { return false }
              } */
              agent {
                dockerfile {
                  additionalBuildArgs '--pull'
                  dir '.CI/cache-arm32'
                  /* The cache Dockerfile makes /cache/runtest, etc world writable
                   * This is necessary because we run the docker image as a user and need to
                   * be able to have a global caching of the omlibrary parts and the runtest database.
                   * Note that the database is stored in a volume on a per-node basis, so the first time
                   * the tests run on a particular node, they might execute slightly slower
                   */
                  label 'linux-arm32'
                  args '--memory=500m --memory-swap=500m ' +
                       "--mount type=volume,source=runtest-omsimulator-cache-arm32,target=/cache/runtest " +
                       '--tmpfs /tmp'
                }
              }
              environment {
                RUNTESTDB = "/cache/runtest/"
              }
              steps {
                sh 'hostname'
                unstash name: 'arm32-install'

                // Work-around for deleting logs
                sh 'find testsuite/ -name "*.lua" -exec sed -i /teardown_command/d {} ";"'

                partest()

                // Disable until working
                // junit 'testsuite/partest/result.xml'
              }
            }
          }
        }

        stage('osxcross') {
          stages {
            stage('cross-compile') {
              agent {
                docker {
                  image 'docker.openmodelica.org/osxcross-omsimulator:v2.0'
                  label 'linux'
                  alwaysPull true
                  args '--tmpfs /tmp'
                }
              }
              environment {
                CROSS_TRIPLE = "x86_64-apple-darwin15"
                CC = "${env.CROSS_TRIPLE}-cc"
                CXX = "${env.CROSS_TRIPLE}-c++"
                AR = "${env.CROSS_TRIPLE}-ar"
                RANLIB = "${env.CROSS_TRIPLE}-ranlib"
                FMIL_FLAGS = '-DFMILIB_FMI_PLATFORM=darwin64'
                detected_OS = 'Darwin'
                VERBOSE = '1'
                BOOST_ROOT = '/opt/osxcross/macports/pkgs/opt/local/'
                OMSFLAGS = "OMTLM=OFF"
              }
              steps {
                buildOMS()
                sh '''
                (cd install/mac && zip -r "../../OMSimulator-osx-`git describe`.zip" *)
                '''

                archiveArtifacts "OMSimulator-osx*.zip"
                stash name: 'osx-install', includes: "install/mac/**"
              }
            }

            stage('test') {
              /* when {
                beforeAgent true
                expression { return false }
              } */
              agent {
                label 'osx'
              }
              steps {
                unstash name: 'osx-install'
                withEnv(["RUNTESTDB=${env.HOME}/jenkins-cache/runtest/"]) {
                  partest()
                }
                junit 'testsuite/partest/result.xml'
              }
            }
          }
        }
        stage('mingw64') { stages {
        stage('mingw64-cross') {
          agent {
            docker {
              image 'docker.openmodelica.org/msyscross-omsimulator:v2.1.0'
              label 'linux'
              alwaysPull true
              args '--tmpfs /tmp'
            }
          }
          environment {
            CROSS_TRIPLE = "x86_64-w64-mingw32"
            CC = "${env.CROSS_TRIPLE}-gcc-posix"
            CXX = "${env.CROSS_TRIPLE}-g++-posix"
            CPPFLAGS = '-I/opt/pacman/mingw64/include'
            LDFLAGS = '-L/opt/pacman/mingw64/lib'
            AR = "${env.CROSS_TRIPLE}-ar-posix"
            RANLIB = "${env.CROSS_TRIPLE}-ranlib-posix"
            detected_OS = 'MINGW64'
            VERBOSE = '1'
            BOOST_ROOT = '/opt/pacman/mingw64/'
            MSYSROOT = '/opt/pacman/'
          }

          steps {
            buildOMS()
            sh '''
            (cd install/mingw && zip -r "../../OMSimulator-mingw64-`git describe`.zip" *)
            '''
            archiveArtifacts "OMSimulator-mingw64*.zip"
            writeFile(file:"install/mingw/bin/OMSimulator", text:"""#!/bin/sh
            export "HOME=${WORKSPACE}"
            export WINEDEBUG=-all
            exec /usr/bin/wine64 "${WORKSPACE}/install/mingw/bin/OMSimulator.exe" "\$@"
            """)
            writeFile(file:"install/mingw/bin/OMSimulatorPython", text:"""#!/bin/sh
            export "HOME=${WORKSPACE}"
            export "PYTHONPATH=${WORKSPACE}/install/mingw/bin;${WORKSPACE}/install/mingw/lib"
            export "WINEPATH=${WORKSPACE}/install/mingw/bin;${WORKSPACE}/install/mingw/lib;/opt/pacman/mingw64/bin"
            export WINEDEBUG=-all
            exec /usr/bin/wine64 /opt/pacman/mingw64/bin/python3.7.exe "\$@"
            """)
            stash name: 'mingw64-install', includes: "install/mingw/**"
            sh "chmod +x install/mingw/bin/OMSimulator install/mingw/bin/OMSimulatorPython"
            sh "install/mingw/bin/OMSimulator --version || install/mingw/bin/OMSimulator --version"
          }
        }
        stage('mingw64-test') {
          agent {
            label 'omdev'
          }
          environment {
            PATH = "${env.PATH};C:\\bin\\git\\bin;C:\\bin\\git\\usr\\bin;C:\\OMDev\\tools\\msys\\mingw64\\bin\\"
            OMDEV = "/c/OMDev"
            MSYSTEM = "MINGW64"
          }

          steps {
            unstash name: 'mingw64-install'
            omdevCommand("install/mingw/bin/OMSimulator.exe --version")
            omdevCommand("make -C testsuite difftool resources")
            omdevCommand("perl ./runtests.pl -nocolour -with-xml ${params.RUNTESTS_FLAG}")
            junit 'testsuite/partest/result.xml'
          }
        }
        } }

        stage('mingw32-cross') {
          when {
            expression { return params.MINGW32 }
            beforeAgent true
          }
          agent {
            docker {
              image 'docker.openmodelica.org/msyscross-omsimulator:v2.1.0-i686'
              label 'linux'
              alwaysPull true
              args '--tmpfs /tmp'
            }
          }
          environment {
            CROSS_TRIPLE = "i686-w64-mingw32"
            CC = "${env.CROSS_TRIPLE}-gcc-posix"
            CXX = "${env.CROSS_TRIPLE}-g++-posix"
            CPPFLAGS = '-I/opt/pacman/mingw32/include'
            LDFLAGS = '-L/opt/pacman/mingw32/lib'
            AR = "${env.CROSS_TRIPLE}-ar-posix"
            RANLIB = "${env.CROSS_TRIPLE}-ranlib-posix"
            detected_OS = 'MINGW32'
            VERBOSE = '1'
            BOOST_ROOT = '/opt/pacman/mingw32/'
            MSYSROOT = '/opt/pacman/'
            OMSFLAGS = "OMTLM=OFF" // OMTLMSimulator project does not support 32-bit MINGW
          }

          steps {
            buildOMS()
            sh '''
            (cd install/mingw && zip -r "../../OMSimulator-mingw32-`git describe`.zip" *)
            '''

            archiveArtifacts "OMSimulator-mingw32*.zip"
            stash name: 'mingw32-install', includes: "install/mingw/**"
          }
        }

        stage('msvc64') {
          when {
            expression { return params.MSVC64 }
            beforeAgent true
          }
          agent {
            label 'omsimulator-windows'
          }
          environment {
            PATH = "${env.PATH};C:\\bin\\git\\bin;C:\\bin\\git\\usr\\bin;C:\\OMDev\\tools\\msys\\mingw64\\bin\\"
            OMDEV = "/c/OMDev"
            MSYSTEM = "MINGW64"
          }

          steps {
            writeFile file: "buildZip.sh", text: """#!/bin/sh
set -x -e
export PATH="/c/Program Files/TortoiseSVN/bin/:/c/bin/jdk/bin:/c/bin/nsis/:\$PATH:/c/bin/git/bin"
cd "${env.WORKSPACE}/install/win"
zip -r "../../OMSimulator-win64-`git describe`.zip" *
"""

            bat """
set BOOST_ROOT=C:\\local\\boost_1_64_0
set PATH=C:\\bin\\cmake\\bin;%PATH%

call configWinVS.bat VS15-Win64
IF NOT ["%ERRORLEVEL%"]==["0"] GOTO fail

call buildWinVS.bat VS15-Win64
IF NOT ["%ERRORLEVEL%"]==["0"] GOTO fail

call install\\win\\bin\\OMSimulator.exe --version
IF NOT ["%ERRORLEVEL%"]==["0"] GOTO fail

C:\\OMDev\\tools\\msys\\usr\\bin\\sh --login -i '${env.WORKSPACE}/buildZip.sh'

EXIT /b 0

:fail
ECHO Something went wrong!
EXIT /b 1
"""

            archiveArtifacts "OMSimulator-win64*.zip"
          }
        }

      }
    }

    stage('upload') {
      parallel {

        stage('upload-doc') {
          when {
            allOf {
              not {
                changeRequest()
              }
              anyOf {
                buildingTag()
                anyOf {
                  branch 'master'
                  branch 'jenkins' // For testing purposes
                  branch 'maintenance/**'
                }
              }
            }
            beforeAgent true
          }
          agent {
            label 'linux'
          }
          /*
          // Does  not pass GIT_BRANCH env.var
          options {
            skipDefaultCheckout()
          }
          */
          steps {
            unstash name: 'docs'
            sh "test ! -z '${env.GIT_BRANCH}'"
            sh "test ! '${env.GIT_BRANCH}' = 'null'"
            sshPublisher(publishers: [sshPublisherDesc(configName: 'OMSimulator-doc', transfers: [sshTransfer(execCommand: "rm -rf .tmp/${env.GIT_BRANCH}"), sshTransfer(execCommand: "test ! -z '${env.GIT_BRANCH}' && rm -rf '/var/www/doc/OMSimulator/${env.GIT_BRANCH}' && mkdir -p `dirname '/var/www/doc/OMSimulator/.tmp/${env.GIT_BRANCH}'` && mv '/var/www/doc/OMSimulator/.tmp/${env.GIT_BRANCH}' '/var/www/doc/OMSimulator/${env.GIT_BRANCH}'", remoteDirectory: ".tmp/${env.GIT_BRANCH}", removePrefix: "install/linux/doc", sourceFiles: 'install/linux/doc/**')])])
          }
        }

      }
    }

  }
}

def numPhysicalCPU() {
  def uname = sh script: 'uname', returnStdout: true
  if (uname.startsWith("Darwin")) {
    return sh (
      script: 'sysctl hw.physicalcpu_max | cut -d" " -f2',
      returnStdout: true
    ).trim().toInteger() ?: 1
  } else {
    return sh (
      script: 'lscpu -p | egrep -v "^#" | sort -u -t, -k 2,4 | wc -l',
      returnStdout: true
    ).trim().toInteger() ?: 1
  }
}

void partest(cache=true, extraArgs='') {
  echo "cache: ${cache}, asan: ${env.ASAN}"
  sh """
  make -C testsuite difftool resources
  cp -f "${env.RUNTESTDB}/"* testsuite/ || true
  """

  // Work-around for deleting logs
  sh 'find testsuite/ -name "*.lua" -exec sed -i /teardown_command/d {} ";"'

  sh ("""#!/bin/bash -x
  ulimit -t 1500
  ${env.ASAN ? "" : "ulimit -v 6291456" /* Max 6GB per process */}

  cd testsuite/partest
  ./runtests.pl ${env.ASAN ? "-asan": ""} -j${numPhysicalCPU()} -nocolour -with-xml ${params.RUNTESTS_FLAG} ${extraArgs}
  CODE=\$?
  test \$CODE = 0 -o \$CODE = 7 || exit 1
  """
  + (cache ?
  """
  if test \$CODE = 0; then
    mkdir -p "${env.RUNTESTDB}/"
    cp ../runtest.db.* "${env.RUNTESTDB}/"
  fi
  """ : ''))
}

void buildOMS() {
  echo "${env.NODE_NAME}"
  def nproc = numPhysicalCPU()
  sh """
  ${env.SHELLSTART ?: ""}
  export HOME="${'$'}PWD"
  git fetch --tags
  make config-3rdParty ${env.CC ? "CC=" + env.CC : ""} ${env.CXX ? "CXX=" + env.CXX : ""} ${env.OMSFLAGS ?: ""} -j${nproc}
  make config-OMSimulator -j${nproc} ${env.ASAN ? "ASAN=ON" : ""} ${env.OMSFLAGS ?: ""}
  make OMSimulator -j${nproc} ${env.ASAN ? "DISABLE_RUN_OMSIMULATOR_VERSION=1" : ""} ${env.OMSFLAGS ?: ""}
  """
}

def omdevCommand(cmd) {
  bat """
  call C:\\OMDev\\tools\\msys\\usr\\bin\\sh --login -c "cd '${env.WORKSPACE.replace("\\","/")}' && ${cmd}"
  EXIT /b %ERRORLEVEL%
  """
}
