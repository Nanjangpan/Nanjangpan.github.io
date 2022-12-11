---
layout: post
title: Airflow Integrition Test (with Pytest)
date: '2022-12-11 10:59:10 +0900'
description: 'Airflow Integrition Test'
categories: [Airflow]
tags: [Airflow]   # TAG names should always be lowercase
---


## 1. 목표

이전에는 코드를 푸쉬한 후에 이상이 있으면 아래의 DAG Import Errors가 나와서 다른 사람들이 해당 DAG를 접근하지 못하는 문제점이 발생하였다. 해당 문제점을 해결하기 위해 Airflow를 프로덕션 레벨로 배포하기 전에 코드 레벨로서 Airflow의 무결성 체크를 하고 싶다.

DAG 무결성 검사는 아래의 dag import error를 잡는 것으로, task 단위의 실패는 못 잡는다.

![image](https://i.ibb.co/s9ys53c/image.png)

## 2. 주어진 환경

-   `airflow==2.3.4`
-   `python==3.7.12`
-   `pytest-cov==4.0.0`
-   `pytest==7.2.0`

## 3. 사전 지식

-   Pytest : [](https://docs.pytest.org/en/7.2.x/)[https://docs.pytest.org/en/7.2.x/](https://docs.pytest.org/en/7.2.x/)
-   Pytest-cov : [](https://readthedocs.org/projects/pytest-cov/)[https://readthedocs.org/projects/pytest-cov/](https://readthedocs.org/projects/pytest-cov/)
-   Monkeypatch : [](https://docs.pytest.org/en/7.1.x/how-to/monkeypatch.html)[https://docs.pytest.org/en/7.1.x/how-to/monkeypatch.html](https://docs.pytest.org/en/7.1.x/how-to/monkeypatch.html)
-   CircleCI : [](https://circleci.com/docs/getting-started/)[https://circleci.com/docs/getting-started/](https://circleci.com/docs/getting-started/)
-   Docker : [](https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html)[https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html](https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html)

## 4. 구현

### 4.1 전체 구조

```bash
├── .cicleci
│   └── config.yaml # PR 단계에서 test를 돌리기 위한 CircleCI 설정 파일
├── test_requirements.txt # test를 위한 dependancy
├── Dockerfile # Airflow docker
├── Dags # dag 파일 있는 곳
├── tests # 테스트
│   ├── dags
│   │   └── test_dag_integrity.py
└── └── conftest.py 
```

![2022-12-11-10-12-05](https://i.ibb.co/tZB0hQZ/2022-12-11-10-12-05.png)

### 4.2 Conftest.py

```python
# conftest.py
import pytest

import sys

assert (
    "airflow" not in sys.modules
), "No airflow module can be imported before these lines"

@pytest.fixture(autouse=True)
def airflow_variables():
    return {
        "variables1": "abc",
        "variables2": "abc",
        "variables3": "abc",
        "variables4": "abc",		
    }

```

-   pytest에서 공통으로 사용되는 Fixture / Plugin / Module을 모아두는 파일이다.
-   Airflow variables를 사용하고 있다면 해당 부분을 mocking용으로 작성
    -   Airflow variables에 예민한 값이 있다면 해당 값을 임의의 문자로 교체
    -   autouse를 true로 설정하여, 별도 요청 없이 모든 테스트 함수에서 해당 fixture를 사용

### 4.3 test_dag_integrity.py

```python
# test_dag_integrity.py
import pytest

import glob
import importlib.util
from airflow.models import DAG, Variable
from airflow.utils.dag_cycle_tester import check_cycle
from pathlib import Path

DIR = Path(__file__).parents[0]
DAG_PATH = DIR / ".." / ".." / "dags/**.py"
DAG_FILES = glob.glob(str(DAG_PATH))

def import_dag_files(dag_path, dag_file):
    module_name = Path(dag_path).stem
    module_path = dag_path / dag_file
    mod_spec = importlib.util.spec_from_file_location(module_name, module_path)
    module = importlib.util.module_from_spec(mod_spec)
    mod_spec.loader.exec_module(module)

    return module

@pytest.mark.parametrize("dag_file", DAG_FILES)
def test_dag_integrity(airflow_variables, dag_file, monkeypatch):
		# Airlfow variables monkey patch
    def mock_get(*args, **kwargs):
        mocked_dict = airflow_variables
        return mocked_dict.get(args[0])

    monkeypatch.setattr(Variable, "get", mock_get)

    module = import_dag_files(DAG_PATH, dag_file)
    dag_objects = [var for var in vars(module).values() if isinstance(var, DAG)]
    assert dag_objects

    for dag in dag_objects:
        check_cycle(dag)

```

-   Airflow 무결성 테스트를 pytest로 돌리기 위해 앞에 이름을 test_로 설정
-   “import_dag_files” ****함수에서 dags 폴더에 있는 모든 python 파일 로드
-   “mock_get” ****와 “monkeypatch” ****를 통해서 Airflow variables moking
-   “check_cycle”와 “dag_objects”를 통해서 무결성 체크
    -   assert ****dag_objects : dag import error가 있는지 검사
    -   check_cycle
        -   code : [](https://github.com/apache/airflow/blob/f3f38a1857ee88573232c7033808c9297c20209f/airflow/utils/dag_cycle_tester.py#L49)[https://github.com/apache/airflow/blob/f3f38a1857ee88573232c7033808c9297c20209f/airflow/utils/dag_cycle_tester.py#L49](https://github.com/apache/airflow/blob/f3f38a1857ee88573232c7033808c9297c20209f/airflow/utils/dag_cycle_tester.py#L49)
        -   airflow task가 A → B → A 처럼 Cycle이 존재하는 검사하고 존재하면*`AirflowDagCycleException`* 발생
        -   BFS를 통해서 Cycle이 있는지 검사

### 4.4 Docker & CircleCI

-   Airflow 테스트 환경 구축을 위한 도커 이미지

```docker
# Dockerfile
FROM <airflow image name>
MAINTAINER jinwoo.baek <jinwoo.baek@linecorp.com>

COPY . /opt/airflow

# no cache dir : <https://stackoverflow.com/questions/45594707/what-is-pips-no-cache-dir-good-for>
RUN pip install --no-cache-dir --upgrade pip \\
  && pip install --no-cache-dir -r test_requirements.txt

WORKDIR /opt/airflow

```

-   test dependacny

```
# test_requirements.txt
pytest==7.2.0
pytest-cov==4.0.0

```

-   CircleCI

```yaml
# .circleci/config.yaml
test:
    docker:
      - image: <airflow image name>
    steps:
      - checkout
      - run:
          name: Install dependencies
          command: |
            pip install --upgrade pip
            pip install -r test_requirements.txt
      - run:
          name: Run pytest
          command: |
            mkdir test-results
            PYTHONPATH=tests coverage run -m pytest -ra -q --junitxml=test-results/junit.xml
            coverage report --omit=tests/*
            coverage html
      - store_test_results: # <https://circleci.com/docs/collect-test-data/#pytest>
          path: test-results
      - store_artifacts: # <https://circleci.com/docs/code-coverage/>
          path: htmlcov

```

## 5. 결과

-   PR을 올릴 때마다 CircleCI가 작동하여 dag 무결성 테스트를 수행하여 이상이 있으면 merge를 못하도록 설정하였다.
    
   ![Untitled](https://i.ibb.co/SKhvYMW/Untitled.png)
    
-   CircleCI 화면
    
![2022-12-11-11-32-58](https://i.ibb.co/XxbZn91/2022-12-11-11-32-58.png)

-   CircleCI test coverage 보고서
    
    ![2022-12-11-11-33-36](https://i.ibb.co/wWkZqC0/2022-12-11-11-33-36.png)
    
    -   ARTIFACTS 탭을 클릭 후 **htmlcov/index.html** 파일 클릭하면 전체 coverage 보고서를 볼 수 있다.

## 6. 출처

-   Apache Airflow best practice : [](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#testing-a-dag)[https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#testing-a-dag](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#testing-a-dag)