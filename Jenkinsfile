#!/usr/bin/env groovy

properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '3', daysToKeepStr: '', numToKeepStr: '5']]]);

def fileName = ""

node {

    def pyarrowVersion = "apache-arrow-13.0.0"

    stage("Build dockerfile") {

        echo "############################################################################"
        echo 'Building Dockerfile images'
        echo "############################################################################"
        
        sh "ls -lah"

        def dockerfileContent = """\
        FROM public.ecr.aws/lambda/python:3.11 AS base

        RUN yum install -y \
            boost-devel \
            jemalloc-devel \
            libxml2-devel \
            libxslt-devel \
            bison \
            make \
            gcc \
            gcc-c++ \
            flex \
            autoconf \
            zip \
            git \
            ninja-build \
            zlib \
            zlib-build \
            zlib-devel 
    
        
        
        RUN pip3 install --upgrade pip wheel
        RUN pip3 install --upgrade urllib3==1.26.16
        RUN pip3 install --upgrade six cython cmake hypothesis poetry
        RUN pip3 install --upgrade setuptools
        WORKDIR /pyarrow-build
        """.stripIndent()
                
        writeFile file: './Dockerfile', text: dockerfileContent

    }

    stage("Build docker") {


        echo "############################################################################"
        echo 'Clone current pyarrow & build wheel'
        echo "############################################################################"

        def customImage = docker.build("pyarrow-image", ".")
        
        customImage.inside('-v $WORKSPACE/output:/output -u root') {

            stage ("clone repo & setup env") {
                sh '''
                    cd /pyarrow-build
                    


                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    export CMAKE_PREFIX_PATH=$ARROW_HOME:$CMAKE_PREFIX_PATH
                    
                    cd $WORKDIR

                    #cleanup
                    rm -rf ./arrow

                    git clone \
                        --depth 1 \
                        --branch apache-arrow-13.0.0 \
                        --single-branch \
                        https://github.com/apache/arrow.git
                    
                    rm -rf $WORKDIR/venv
                    
                    python -m venv $WORKDIR/venv
                    . $WORKDIR/venv/bin/activate
                    
                    pip install --upgrade six cython cmake hypothesis poetry
                    pip install -r $ARROW_ROOT/python/requirements-wheel-build.txt
                    pip install setuptools_scm==7.1.0
                '''
            }

            stage ("build c++ lib") {
                sh '''
                    set -e

                    cd /pyarrow-build

                    ls -lah
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    export CMAKE_PREFIX_PATH=$ARROW_HOME:$CMAKE_PREFIX_PATH

                    . $WORKDIR/venv/bin/activate
                    

                    #----------------------------------------------------------------------
                    # Build C++ library
                    
                    rm -rf $CPP_BUILD_DIR

                    mkdir -p $CPP_BUILD_DIR
                    cd $CPP_BUILD_DIR
                    
                    mkdir /usr/lib/x86_64-linux-gnu
                    
                    NINJA="ninja-build"
                    
                    #ln -s /usr/bin/ninja-build /usr/bin/ninja 
                    ls -lah /usr/lib
                    ln -s /usr/lib64/libz.so.1 /usr/lib/x86_64-linux-gnu/libz.so
                   
                    cmake \
                        -DCMAKE_INSTALL_PREFIX=$ARROW_HOME \
                        -DCMAKE_INSTALL_LIBDIR=lib \
                        -DARROW_PYTHON=ON \
                        -DARROW_PARQUET=ON \
                        -DARROW_DATASET=ON \
                        -DARROW_WITH_SNAPPY=ON \
                        -DARROW_WITH_ZLIB=ON \
                        -DARROW_FLIGHT=OFF \
                        -DARROW_GANDIVA=OFF \
                        -DARROW_ORC=OFF \
                        -DARROW_CSV=ON \
                        -DARROW_JSON=ON \
                        -DARROW_COMPUTE=ON \
                        -DARROW_FILESYSTEM=ON \
                        -DARROW_PLASMA=OFF \
                        -DARROW_WITH_BZ2=OFF \
                        -DARROW_WITH_ZSTD=OFF \
                        -DARROW_WITH_LZ4=OFF \
                        -DARROW_WITH_BROTLI=OFF \
                        -DARROW_BUILD_TESTS=OFF \
                        -GNinja \
                        $ARROW_ROOT/cpp
                        
                    #ninja install
                    
                    eval $NINJA
                    eval "${NINJA} install"
                    
                    
                    
                    cd $WORKDIR
                '''
            }

            stage ("build pyhthon lib") {
                sh '''
                    set -e

                    cd /pyarrow-build
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    

                    cd $WORKDIR

                    . ./venv/bin/activate
            
                    #----------------------------------------------------------------------
                    # Build and test Python library
                    cd $ARROW_ROOT/python
                    
                    
                    rm -rf build/  # remove any pesky pre-existing build directory
                    
                    export CMAKE_PREFIX_PATH=${ARROW_HOME}${CMAKE_PREFIX_PATH:+:${CMAKE_PREFIX_PATH}}
                    export ARROW_PRE_0_15_IPC_FORMAT=0
                    export PYARROW_WITH_HDFS=0
                    export PYARROW_WITH_FLIGHT=0
                    export PYARROW_WITH_GANDIVA=0
                    export PYARROW_WITH_ORC=0
                    export PYARROW_WITH_CUDA=0
                    export PYARROW_WITH_PLASMA=0
                    export PYARROW_WITH_PARQUET=1
                    export PYARROW_WITH_DATASET=1
                    export PYARROW_WITH_FILESYSTEM=1
                    export PYARROW_WITH_CSV=0
                    export PYARROW_WITH_JSON=0
                    export PYARROW_WITH_COMPUTE=1
                    export PYARROW_CMAKE_GENERATOR=Ninja

                    #python -m build build_ext --build-type=release --bundle-arrow-cpp
                    python setup.py build_ext --build-type=release --bundle-arrow-cpp bdist_wheel
                '''
            }


            stage ("coping files") {
                sh '''
                    set -e

                    ls -lah

                    cd /pyarrow-build
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    
                    . $WORKDIR/venv/bin/activate

                    cd $ARROW_ROOT/python


                    rm -f /output/pyarrow-*.whl

                    mv ./dist/pyarrow-*.whl /output/


                    rm -rf /output/pyarrow-files/

                    mkdir /output/pyarrow-files/

                    python -m pip install /output/pyarrow-*.whl -t /output/pyarrow-files/

                    chmod -R 777 $WORKDIR/

                    chmod -R 777 /output/pyarrow-files/
                '''
            }

            

        
        }
    }

    stage("Cleanup files & zip") {
        echo "############################################################################"
        echo 'Cleanup files'
        echo "############################################################################"

        def wheel_build = sh (script: "cd output && find . -print | grep -i '.*[.]whl'", returnStdout: true)

        fileName = wheel_build.replace("whl", "+").replace("\n", "").replace("./", "")+"lambda-minimal-0.4.zip"

        sh (script: '''
        ls -lah
        find . -print | grep -i '.*[.]whl'
        cd output
        ls -lah
        find pyarrow-files -name '*.so' -type f -exec strip "{}" \\;
        find pyarrow-files -wholename "*/tests/*" -type f -delete
        find pyarrow-files -regex '^.*\\(__pycache__\\|\\.py[co]\\)$' -delete
        rm -rf ../python
        mkdir ../python
        mv pyarrow-files/pyarrow* ../python''', returnStdout: true)
        zip zipFile: fileName, archive: false, dir: './python',  overwrite: true
        archiveArtifacts artifacts: "${fileName}"
        archiveArtifacts artifacts: 'output/pyarrow-*.whl'
        
    }
    
}
