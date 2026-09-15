FROM ubuntu:22.04
ENV PYSPARK_PYTHON=python3
ENV PYSPARK_DRIVER_PYTHON=python3
ENV DEBIAN_FRONTEND=noninteractive
ENV HARDWARE_TARGET="cpu"
ENV TF_CPP_MIN_LOG_LEVEL='3'
ENV MODEL_TO_LOAD="image_deid_multi_model_context_pipeline_cpu"
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64/
RUN echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64/" >> ~/.bashrc
ENV LC_ALL=C.UTF-8 LANG=C.UTF-8


RUN apt-get update && apt-get install --no-install-recommends -y \
         python3 \
         python3-dev \
         python3-pip \
         python3-gdcm \
         libgdcm-tools \
         libgl1 \
         wget \
         openjdk-17-jdk-headless \
         git-core \
         curl && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
         && mkdir -p /app/model_cache


# Install Johnsnowlabs libraries
RUN pip config set global.no-cache-dir 1 && pip install packaging johnsnowlabs==6.4.0 fastapi torchvision==0.13.1 pillow_jpls pylibjpeg[all] pydicom pyjpegls pylibjpeg-openjpeg pylibjpeg highdicom boto3 imagecodecs

COPY app.py installer.py /app/
RUN --mount=type=secret,id=license python3 /app/installer.py && rm -rf /root/.javacpp/* /tmp/*

# spark-ocr rc7 + race-fix jar from commit 8d6f54d4 (build with sbt, place here)
RUN pip install spark-ocr==6.4.3rc7 --extra-index-url https://pypi.johnsnowlabs.com/6.4.3rc7-86ccb9191df33a1ccd1c1b0f794b15e7803ae263

COPY spark-ocr-assembly-6.4.3rc7.jar /root/.johnsnowlabs/java_installs/spark-ocr-assembly-6.4.3rc7.jar
RUN python3 - <<'PYEOF'
import json
p = "/root/.johnsnowlabs/java_installs/info.json"
d = json.load(open(p))
def repl(o):
    if isinstance(o, dict):
        for k in list(o):
            if k.startswith("spark-ocr-assembly-"):
                del o[k]
                o["spark-ocr-assembly-6.4.3rc7.jar"] = {
                    "file_name": "spark-ocr-assembly-6.4.3rc7.jar",
                    "product": "Spark-OCR",
                    "compatible_spark_version": "3.x.x",
                    "product_version": "6.4.3rc7",
                    "install_type": "cpu",
                }
                return True
        return any(repl(v) for v in o.values())
    if isinstance(o, list):
        return any(repl(it) for it in o)
    return False
assert repl(d), "no spark-ocr-assembly-*.jar entry found in info.json"
json.dump(d, open(p, "w"), indent=2)
PYEOF

RUN pip install --upgrade pillow

WORKDIR /app
ENV PATH=/opt/program:${PATH}
ENTRYPOINT ["python3", "app.py"]
