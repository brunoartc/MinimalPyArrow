#!/usr/bin/env groovy
node {

    def pyarrowVersion = "apache-arrow-13.0.0"

    stage("Build dockerfile") {

        echo "############################################################################"
        echo 'Building Dockerfile images'
        echo "############################################################################"
        
        def dockerfileContent = """\
        FROM ubuntu:focal

        ENV DEBIAN_FRONTEND=noninteractive

        RUN apt-get update -y -q && \
            apt-get install -y -q --no-install-recommends \
                apt-transport-https \
                software-properties-common \
                wget && \
                add-apt-repository ppa:deadsnakes/ppa &&\
                apt-get install -y -q --no-install-recommends python3.11-dev python3.11-venv &&\
            apt-get install -y -q --no-install-recommends \
            build-essential \
            cmake \
            git \
            ninja-build \
            python3-dev \
            python3-pip \
            python3-venv \
            && \
            apt-get clean && rm -rf /var/lib/apt/lists*

        RUN pip3 install -U pip setuptools

        RUN wget -O - https://bootstrap.pypa.io/get-pip.py | python3.11""".stripIndent()
                
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
                    set -e

                    ls -lah
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    

                    #cleanup
                    rm -rf ./arrow

                    git clone \
                        --depth 1 \
                        --branch apache-arrow-12.0.0 \
                        --single-branch \
                        https://github.com/apache/arrow.git
                    
                    python3 -m venv $WORKDIR/venv
                    . $WORKDIR/venv/bin/activate
                    
                    
                    pip install -r $ARROW_ROOT/python/requirements-wheel-build.txt
                '''
            }

            stage ("build c++ lib") {
                sh '''
                    set -e

                    ls -lah
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH

                    . $WORKDIR/venv/bin/activate
                    

                    #----------------------------------------------------------------------
                    # Build C++ library
                    
                    mkdir -p $CPP_BUILD_DIR
                    cd $CPP_BUILD_DIR
                    
                    cmake -GNinja \
                        -DCMAKE_BUILD_TYPE=MinSizeRel \
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
                        -DARROW_CSV=OFF \
                        -DARROW_JSON=OFF \
                        -DARROW_COMPUTE=ON \
                        -DARROW_FILESYSTEM=ON \
                        -DARROW_PLASMA=OFF \
                        -DARROW_WITH_BZ2=OFF \
                        -DARROW_WITH_ZSTD=OFF \
                        -DARROW_WITH_LZ4=OFF \
                        -DARROW_WITH_BROTLI=OFF \
                        -DARROW_BUILD_TESTS=OFF \
                        $ARROW_ROOT/cpp
                    
                    ninja install
                    
                    cd $WORKDIR
                '''
            }

            stage ("build pyhthon lib") {
                sh '''
                    set -e

                    ls -lah
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    

                    cd $WORKDIR

                    . $WORKDIR/venv/bin/activate
            
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
                    
                    # You can run either "develop" or "build_ext --inplace". Your pick
                    
                    python setup.py build_ext --build-type=release --bundle-arrow-cpp bdist_wheel
                    # python setup.py develop
                '''
            }


            stage ("coping files") {
                sh '''
                    set -e

                    ls -lah
                    
                    WORKDIR=$(pwd)
                    CPP_BUILD_DIR=$WORKDIR/arrow-cpp-build
                    ARROW_ROOT=$WORKDIR/arrow
                    export ARROW_HOME=$WORKDIR/dist
                    export LD_LIBRARY_PATH=$ARROW_HOME/lib:$LD_LIBRARY_PATH
                    
                    . $WORKDIR/venv/bin/activate

                    cd $ARROW_ROOT/python
                    
                    find . -print | grep -i '.*[.]whl'

                    rm -rf /output/pyarrow-files/

                    rm /output/pyarrow-*.whl

                    mv ./dist/pyarrow-*.whl /output/

                    mkdir /output/pyarrow-files/
                    
                    rm -rf output/pyarrow-files/*

                    python -m pip install /output/pyarrow-*.whl -t /output/pyarrow-files/

                    chmod -R 777 /output/pyarrow-files/
                '''
            }

            

        
        }
    }

    stage("Cleanup files & zip") {
        echo "############################################################################"
        echo 'Cleanup files'
        echo "############################################################################"

        def fileName = pyarrowVersion+"-minimal-0.4.zip"

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
        archiveArtifacts artifacts: fileName
        
    }
    
}
