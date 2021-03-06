# Analysis Module

- [Introduce](#introduce)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
    - [From Source](#from-source)
    - [Docker Compose](#docker-compose)
- [Setting Module](#setting-module)
    - [Configure Module Class](#configure-module-class)
    - [Modify Tasks](#modify-tasks)
    - [Additional Settings](#additional-settings)
- [Setting Database](#setting-database)
- [Run Web Server](#run-web-server)
    
## Introduce

본 프로젝트는 Neural Network의 결과를 REST API로 서비스 하기 위한 웹 서버를 제공합니다.

Python 코드로 구성되어 있으며, Django 및 Django REST framework를 사용하여 개발하였습니다.

본 프로젝트는 [Analysis Site](https://github.com/sogang-mm/analysis-site)와 함께 설치하기를 추천합니다.

Linux 사용을 가정하여 코드를 작성하였으며, 만약 다른 환경에서의 설치를 진행하려면 문의하시기 바랍니다.

프로젝트의 개발은 docker container에서 진행하시기를 권장하며, host PC에서 바로 개발을 진행할 경우 mysql 설치 및 아래 DB, 계정정보를 설정해야 하니 필요하신 경우 맨아래 메일로 문의히시기 바랍니다.
```
# DB info
Database name : module_db
User : admin
password : password
```

## Prerequisites

- Linux Based OS
- Python 3
- And so on


## Installation

### From Source

실행에 필요한 service를 설치합니다.
```bash
sudo apt-get install rabbitmq-server
sudo service rabbitmq-server restart
```

실행에 필요한 package를 설치합니다.
```bash
pip install -r requirements.txt
```

만약 package 설치가 진행되지 않는다면 pip를 업데이트 한 후 다시 시도합니다.
```bash
pip install --upgrade pip
pip install setuptools
```

### Docker Compose

Docker Compose를 사용하기 위해서는 다음을 필요로 합니다.

- [Docker](https://docs.docker.com/) & [Docker compose](https://docs.docker.com/compose/)
- [NVIDIA Container Runtime for Docker](https://github.com/NVIDIA/nvidia-docker)

이후, 디렉토리 내에서 다음과 같은 부분을 수정합니다.

1. Dockerfile
    * 본인이 사용할 Deep learning framework가 담긴 Docker image로 수정합니다.
    ```dockerfile
    FROM ubuntu:16.04
    ```
    * 수정가능한 docker image는 아래와 같으며 원하는 docker image가 없을 경우 맨아래 문의 메일로 문의주시기 바랍니다.
    ```dockerfile
    FROM sogangmm/cuda:10.2-cudnn7-devel-ubuntu18.04-py36-mysql
    ```

2. docker-compose.yml
    * Module의 외부 통신을 위한 Port 수정이 필요하다면 다음을 수정합니다.
    ```docker
    ports:
      - "8001:8000"
    ```
    * 앞의 8001번을 원하는 포트로 수정한다. 예를 들어 8002번 포트로 접속하기 원한다면 "8002:8000"로 수정합니다.

3. docker-compose-env/main.env
    * 특정 GPU만 사용하는 환경을 구성하고 싶다면 다음을 수정합니다.
    ```text
    NVIDIA_VISIBLE_DEVICES=all
    ```    
    * all을 사용 시, 전체 GPU를 사용한다. 만약 0번 GPU만을 사용하고 싶다면 NVIDIA_VISIBLE_DEVICES=0으로 수정합니다.

모든 설정이 끝났다면 docker 디렉토리 내에서 docker-compose up -d으로 실행하면 웹 서버가 시작됩니다.

http://localhost:8001/ 또는 구성한 서버의 IP 및 Domain으로 접근하여 접속이 되는지 확인합니다.

웹 서버가 실행된 것을 확인하였으면 Module 추가를 위해 main container에 docker attach로 접근하여 일단 웹 서버를 종료합니다.

```bash
docker attach analysis-module-v2_main_1
Ctrl + C
sh server_shutdown.sh
```

Docker container에 ssh 로 접속하고 싶은 경우, 아래와 같이 계정의 password를 설정하고 ssh service를 시작한다.
```bash
passwd
service ssh start
```
 

## Setting Module

모든 설치가 끝났다면 Modules을 추가하기 위해 Modules 디렉토리로 이동합니다.
여기에는 작성에 도움을 주기 위해 dummy 디렉토리 내 main.py를 참고하여 작성합니다.<br>
각각의 모듈별로 필요한 input type(video, audio, text)에 따라 아래와 같이 dummy 디렉토리 내 main.py를 상속 및 참고하여 작성합니다.br>
Dummy class에는 input type에 따라 함수가 정의되어 있으며(inference_by_image, inference_by_audio, inference_by_text), video의 경우 inference_by_video를 반복적으로 호출하여 inference를 진행하기 때문에 inference_by_image를 수정해야 합니다.

### Configure Module Class

* Module 내 다른 python import 하기
    ```python
    from Modules.dummy.example import test
    ```
    * Django 실행 시 root 폴더가 프로젝트의 최상위 폴더가 되므로, sub 폴더 내 다른 python 파일을 import 위해서는 위와 같이 최상위 폴더 기준으로 import를 해야 합니다.

* \__init\__ 함수
    ```python
    model_path = os.path.join(self.path, "model.txt")
    self.model = open(model_path, "r")
    ```
   * \__init\__에서는 model 불러오기 및 대기 상태 유지를 위한 코드를 작성합니다.
   * 데이터를 분석할 때마다 model을 호출하지 않도록 이 부분에서 model을 load하도록 작성합니다.
   * model 등의 파일을 불러오기 위해선 model_path를 사용하여 절대경로로 불러오도록 합니다.

* inference_by_${type} 함수
    ```python
    # Image
    result = {"frame_result": [
            {
                # 1 bbox & multiple object
                'label': [
                    {'description': 'person', 'score': 1.0},
                    {'description': 'chair', 'score': 1.0}
                ],
                'position': {
                    'x': 0.0,
                    'y': 0.0,
                    'w': 0.0,
                    'h': 0.0
                }
            },
            {
                # 1 bbox & 1 object
                'label': [
                    {'description': 'car', 'score': 1.0},
                ],
                'position': {
                    'x': 100.0,
                    'y': 100.0,
                    'w': 100.0,
                    'h': 100.0
                }
            }
        ]}
    # audio
    result = {"audio_result": [
            {
                # 1 timestamp & multiple class
                'label': [
                    {'score': 1.0, 'description': 'class_name'},
                    {'score': 1.0, 'description': 'class_name'}
                ],
                'timestamp': "00:00:01:00"
            },
            {
                # 1 timestamp & 1 class
                'label': [
                    {'score': 1.0, 'description': 'class_name'}
                ],
                'timestamp': "00:00:01:00"
            }
        ]}
    # text
    result = {"text_result": [
            {
                # 1 timestamp & multiple class
                'label': [
                    {'score': 1.0, 'description': 'word_name'},
                    {'score': 1.0, 'description': 'word_name'}
                ],
            },
            {
                # 1 timestamp & 1 class
                'label': [
                    {'score': 1.0, 'description': 'word_name'}
                ],
            }
        ]}
    ```
    * analysis_by_image 함수와 analysis_by_audio는 파일의 경로를 입력으로 받고, analysis_by_text는 text를 받으며 \__init\__ 에서 불러온 모델을 통해 분석 결과를 반환하여 저장합니다. 이때 결과값은 위와 같은 형태를 가지도록 구성합니다.
    * 결과 값에 대한 format을 변경해야 하거나 문의가 있을경우 맨아래 문의 메일로 문의해주시기 바랍니다.

### Modify Tasks

위와 같이 Moduele 설정이 끝났다면 작성한 Module을 추가하기 위해 WebAnalyzer 디렉토리로 이동한다. 그 후 tasks.py를 수정합니다.

* Module 불러오기
    ```python
    @worker_process_init.connect
    def module_load_init(**__):
        global analyzer
        worker_index = current_process().index
    
        print("====================")
        print(" Worker Id: {0}".format(worker_index))
        print("====================")
    
        # TODO:
        #   - Add your model
        #   - You can use worker_index if you need to get and set gpu_id
        #       - ex) gpu_id = worker_index % TOTAL_GPU_NUMBER
        from Modules.dummy.main import Dummy
        analyzer = Dummy()
    ```
    * 위에서 작성한 class을 불러온 후, anlyzer에 추가합니다.
    * 만약 이 때, Multi-gpu를 사용하여 gpu 별로 나누어 추가하고 싶다면, worker_index를 사용하여 이를 수정할 수 있습니다.

* Module 실행하기
    ```python
    @app.task
    def analyzer_by_path(image_path):
        result = analyzer.inference_by_path(image_path)
        return result
    ```
    * 위에서 불러온 Module이 실제로 실행되는 부분으로, 분석 결과를 받아 반환합니다.


### Additional Settings

실행 시에 필요한 다양한 Setting을 변경하고 싶다면 AnalysisModule 디렉토리의 config.py를 수정합니다.

* 개발모드 해제하기
```python
DEBUG = False
```

* 불러오는 Module 수 조절하기
```python
TOTAL_NUMBER_OF_MODULES = 2
```

## Setting Database

### Migration
Django 내 필요한 model 구조를 반영하기 위해 다음을 실행합니다.
```bash
sh run_migration.sh
```
만약 필요에 의해 model 구조를 변경하였다면, run_migration.sh을 통해 생성된 파일을 지우고 다시 설정해주어야 합니다.
```bash
sudo rm db.sqlite3
sh server_initialize.sh
sh run_migration.sh
```

## Run Web Server

* Web Server를 실행하고자 한다면 server_start.sh를 실행합니다.
    ```bash
    sh server_start.sh
    ```
    이후 http://localhost:8001/ 또는 구성한 서버의 IP 및 Domain으로 접근하여 접속합니다.

* 만약 접속 시 문제가 있어 실행 Log를 보고자 할 때는 다음과 같이 실행하여 확인합니다.
    * Web Server에 문제가 있어 Django 부분만 실행하고자 한다면 run_django.sh를 실행합니다.
        ```bash
        sh run_django.sh
        ```
    
    * Web Server는 실행되나 분석 결과가 나오지 않아 Module 부분만 실행하고자 한다면 run_celery.sh를 실행합니다.
        ```bash
        sh run_celery.sh
        ```
    
* Web Server를 종료하고자 한다면 server_shutdown.sh를 실행합니다.
    ```bash
    sh server_shutdown.sh
    ``` 

## Contact
Email : [jinhasong@sogang.ac.kr](jinhasong@sogang.ac.kr)
Phone : 010-4014-8730
