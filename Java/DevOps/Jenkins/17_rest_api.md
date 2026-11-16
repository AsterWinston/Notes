
# 1. API 概述

Jenkins 提供丰富的 REST API，用于外部系统集成和自动化管理。API 遵循 RESTful 风格，支持 JSON 和 XML 格式。

**API 基础 URL**：`http://jenkins:8080/api/`

**支持的格式**：
- JSON：`/api/json`
- XML：`/api/xml`
- Python：`/api/python`

# 2. 认证方式

## 2.1 基本认证

```bash
# 方式一：用户名 + API Token
curl -u username:api_token http://jenkins:8080/api/json

# 方式二：用户名 + 密码（不推荐）
curl -u username:password http://jenkins:8080/api/json

# 方式三：Bearer Token（需插件）
curl -H "Authorization: Bearer api_token" http://jenkins:8080/api/json
```

## 2.2 API Token 管理

```bash
# 通过 UI 创建 Token
用户 > 设置 > API Token > Add new Token

# 通过 API 创建 Token
curl -X POST -u username:password \
  http://jenkins:8080/user/username/descriptorByName/jenkins.security.ApiTokenProperty/generateToken \
  --data 'newTokenName=my-token'

# 响应：
# {"status":"ok","data":{"tokenName":"my-token","tokenUuid":"xxx","tokenValue":"11a0..."}}
```

## 2.3 CSRF Token

```bash
# 获取 CSRF Token
CRUMB=$(curl -s -u "username:api_token" \
  "http://jenkins:8080/crumbIssuer/api/json" | \
  jq -r '.crumb')

# 使用 CSRF Token 发起 POST 请求
curl -X POST -u "username:api_token" \
  -H "Jenkins-Crumb: $CRUMB" \
  http://jenkins:8080/job/myjob/build
```

# 3. 核心 API 端点

## 3.1 系统信息

```bash
# 获取 Jenkins 系统信息
curl -u username:api_token http://jenkins:8080/api/json?pretty=true

# 响应示例
{
  "assignedLabels": [{"name": "master"}],
  "mode": "NORMAL",
  "nodeDescription": "the master Jenkins node",
  "nodeName": "",
  "numExecutors": 2,
  "description": null,
  "jobs": [
    {"name": "my-pipeline", "url": "http://jenkins:8080/job/my-pipeline/", "color": "blue"}
  ],
  "overallLoad": {},
  "primaryView": {"name": "all", "url": "http://jenkins:8080/"},
  "quietingDown": false,
  "slaveAgentPort": 50000,
  "unlabeledLoad": {},
  "useCrumbs": true,
  "useSecurity": true,
  "views": [
    {"name": "all", "url": "http://jenkins:8080/"}
  ]
}
```

## 3.2 Job 管理

```bash
# 获取所有 Job 列表
curl -u username:token http://jenkins:8080/api/json?tree=jobs[name,url,color]

# 获取单个 Job 详情
curl -u username:token http://jenkins:8080/job/my-pipeline/api/json?pretty=true

# 响应示例
{
  "_class": "org.jenkinsci.plugins.workflow.job.WorkflowJob",
  "name": "my-pipeline",
  "url": "http://jenkins:8080/job/my-pipeline/",
  "buildable": true,
  "builds": [
    {"number": 5, "url": "http://jenkins:8080/job/my-pipeline/5/"},
    {"number": 4, "url": "http://jenkins:8080/job/my-pipeline/4/"}
  ],
  "color": "blue",
  "firstBuild": {"number": 1},
  "healthReport": [
    {"description": "Build stability: 1 out of the last 5 builds failed.", "score": 80}
  ],
  "inQueue": false,
  "keepDependencies": false,
  "lastBuild": {"number": 5, "url": "..."},
  "lastCompletedBuild": {"number": 5},
  "lastFailedBuild": {"number": 3},
  "lastStableBuild": {"number": 5},
  "lastSuccessfulBuild": {"number": 5},
  "lastUnstableBuild": {"number": 2},
  "lastUnsuccessfulBuild": {"number": 3},
  "nextBuildNumber": 6,
  "property": [],
  "queueItem": null,
  "concurrentBuild": false,
  "disabled": false
}

# 创建 Job（通过 XML）
curl -X POST -u username:token \
  -H "Content-Type: application/xml" \
  --data-binary @config.xml \
  http://jenkins:8080/createItem?name=new-job

# 复制 Job
curl -X POST -u username:token \
  http://jenkins:8080/createItem?name=new-job&mode=copy&from=template-job

# 删除 Job
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/doDelete

# 禁用 Job
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/disable

# 启用 Job
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/enable
```

## 3.3 构建管理

```bash
# 触发构建（无参数）
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/build

# 触发构建（带参数）
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/buildWithParameters \
  --data 'BRANCH=feature/test&ENV=dev&SKIP_TESTS=false'

# 触发构建（带 JSON 参数）
curl -X POST -u username:token \
  -H "Content-Type: application/json" \
  --data '{"parameter": [{"name":"BRANCH", "value":"main"}, {"name":"ENV", "value":"prod"}]}' \
  http://jenkins:8080/job/my-pipeline/build

# 获取构建详情
curl -u username:token \
  http://jenkins:8080/job/my-pipeline/5/api/json?pretty=true

# 响应示例
{
  "_class": "org.jenkinsci.plugins.workflow.job.WorkflowRun",
  "actions": [
    {"_class": "hudson.model.CauseAction", "causes": [{"_class": "...UserIdCause", "userId": "admin"}]}
  ],
  "artifacts": [
    {"fileName": "myapp.jar", "relativePath": "target/myapp.jar"}
  ],
  "building": false,
  "description": null,
  "duration": 120000,
  "estimatedDuration": 115000,
  "executor": null,
  "fullDisplayName": "my-pipeline #5",
  "id": "5",
  "keepLog": false,
  "number": 5,
  "queueId": 42,
  "result": "SUCCESS",
  "timestamp": 1700000000000,
  "url": "http://jenkins:8080/job/my-pipeline/5/",
  "changeSets": [],
  "culprits": [],
  "previousBuild": {"number": 4},
  "nextBuild": null
}

# 获取构建日志
curl -u username:token \
  http://jenkins:8080/job/my-pipeline/5/logText/progressiveText

# 下载构建产物
curl -u username:token -O \
  http://jenkins:8080/job/my-pipeline/5/artifact/target/myapp.jar

# 停止构建
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/5/stop

# 删除构建
curl -X POST -u username:token \
  http://jenkins:8080/job/my-pipeline/5/doDelete
```

## 3.4 Pipeline 阶段信息

```bash
# 获取 Pipeline 阶段详情
curl -u username:token \
  http://jenkins:8080/job/my-pipeline/5/wfapi/describe

# 获取 Pipeline 执行状态
curl -u username:token \
  http://jenkins:8080/job/my-pipeline/5/wfapi/

# 获取 Pipeline 节点详情
curl -u username:token \
  http://jenkins:8080/job/my-pipeline/5/flowGraphTable/ajax/
```

## 3.5 节点/Agent 管理

```bash
# 获取所有节点
curl -u username:token http://jenkins:8080/computer/api/json?pretty=true

# 获取节点详情
curl -u username:token http://jenkins:8080/computer/agent-1/api/json

# 创建节点
curl -X POST -u username:token \
  -H "Content-Type: application/xml" \
  --data-binary @node-config.xml \
  "http://jenkins:8080/computer/createItem?name=agent-1"

# 删除节点
curl -X POST -u username:token \
  http://jenkins:8080/computer/agent-1/doDelete

# 离线节点
curl -X POST -u username:token \
  --data 'offlineMessage=Maintenance' \
  http://jenkins:8080/computer/agent-1/doMarkOffline

# 上线节点
curl -X POST -u username:token \
  http://jenkins:8080/computer/agent-1/doMarkOnline
```

## 3.6 队列管理

```bash
# 获取队列信息
curl -u username:token http://jenkins:8080/queue/api/json?pretty=true

# 取消队列中的任务
curl -X POST -u username:token \
  http://jenkins:8080/queue/cancelItem?id=42
```

## 3.7 凭据管理

```bash
# 获取凭据列表
curl -u username:token \
  http://jenkins:8080/credentials/store/system/domain/_/api/json

# 创建凭据
curl -X POST -u username:token \
  -H "Content-Type: application/xml" \
  --data-binary @credential.xml \
  "http://jenkins:8080/credentials/store/system/domain/_/createCredentials"

# credential.xml 内容
# <com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl>
#   <scope>GLOBAL</scope>
#   <id>my-credentials</id>
#   <description>My Credentials</description>
#   <username>myuser</username>
#   <password>mypassword</password>
# </com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl>

# 删除凭据
curl -X POST -u username:token \
  "http://jenkins:8080/credentials/store/system/domain/_/credential/my-credentials/doDelete"
```

## 3.8 插件管理

```bash
# 获取已安装插件列表
curl -u username:token \
  http://jenkins:8080/pluginManager/api/json?depth=1

# 安装插件
curl -X POST -u username:token \
  -H "Content-Type: application/xml" \
  --data '<install plugin="git@latest"/>' \
  http://jenkins:8080/pluginManager/installNecessaryPlugins

# 卸载插件
curl -X POST -u username:token \
  http://jenkins:8080/pluginManager/plugin/git/doUninstall
```

# 4. 高级 API 用法

## 4.1 使用 Groovy 脚本

```bash
# 执行 Groovy 脚本
curl -X POST -u username:token \
  --data-urlencode 'script=println Jenkins.instance.pluginManager.plugins.size()' \
  http://jenkins:8080/scriptText

# 批量操作
curl -X POST -u username:token \
  --data-urlencode 'script=
    Jenkins.instance.allItems(hudson.model.Job).each { job ->
      println "Job: ${job.fullName} - ${job.lastBuild?.result}"
    }
  ' \
  http://jenkins:8080/scriptText
```

## 4.2 过滤和选择

```bash
# 使用 tree 参数选择性获取字段
curl -u username:token \
  "http://jenkins:8080/api/json?tree=jobs[name,url,color,lastBuild[number,result,duration]]"

# 使用 depth 参数控制嵌套深度
curl -u username:token \
  "http://jenkins:8080/api/json?depth=2&pretty=true"

# 使用 xpath 过滤（仅 XML）
curl -u username:token \
  -H "Accept: application/xml" \
  "http://jenkins:8080/api/xml?xpath=/hudson/job[name='my-pipeline']/color"
```

## 4.3 Webhook 回调

```groovy
// Pipeline 中使用 REST API 回调
stage('External Notification') {
    steps {
        script {
            def response = httpRequest(
                url: 'https://api.external-system.com/jenkins-callback',
                httpMode: 'POST',
                contentType: 'APPLICATION_JSON',
                requestBody: groovy.json.JsonOutput.toJson([
                    job: env.JOB_NAME,
                    buildNumber: env.BUILD_NUMBER,
                    status: currentBuild.result ?: 'RUNNING',
                    url: env.BUILD_URL
                ])
            )
            echo "Callback response: ${response.content}"
        }
    }
}
```

# 5. Python 客户端示例

```python
# jenkins_client.py
import requests
import json
from urllib.parse import quote

class JenkinsClient:
    def __init__(self, url, username, token):
        self.url = url.rstrip('/')
        self.auth = (username, token)
        self.session = requests.Session()
        self.session.auth = self.auth
        self.session.headers.update({
            'Content-Type': 'application/json'
        })

    def get_crumb(self):
        """获取 CSRF Token"""
        resp = self.session.get(f'{self.url}/crumbIssuer/api/json')
        return resp.json()['crumb']

    def get_jobs(self):
        """获取所有 Job"""
        resp = self.session.get(f'{self.url}/api/json?tree=jobs[name,url,color]')
        return resp.json().get('jobs', [])

    def get_job_info(self, job_name):
        """获取 Job 详情"""
        resp = self.session.get(
            f'{self.url}/job/{quote(job_name)}/api/json'
        )
        return resp.json()

    def build_job(self, job_name, parameters=None):
        """触发构建"""
        crumbs = {'Jenkins-Crumb': self.get_crumb()}
        if parameters:
            resp = self.session.post(
                f'{self.url}/job/{quote(job_name)}/buildWithParameters',
                headers=crumbs,
                data=parameters
            )
        else:
            resp = self.session.post(
                f'{self.url}/job/{quote(job_name)}/build',
                headers=crumbs
            )
        return resp.status_code == 201

    def get_build_info(self, job_name, build_number):
        """获取构建详情"""
        resp = self.session.get(
            f'{self.url}/job/{quote(job_name)}/{build_number}/api/json'
        )
        return resp.json()

    def get_build_log(self, job_name, build_number):
        """获取构建日志"""
        resp = self.session.get(
            f'{self.url}/job/{quote(job_name)}/{build_number}/logText/progressiveText'
        )
        return resp.text

    def list_builds(self, job_name, limit=10):
        """列出最近的构建"""
        info = self.get_job_info(job_name)
        builds = info.get('builds', [])[:limit]
        for build in builds:
            build['result'] = self.get_build_info(
                job_name, build['number']
            ).get('result')
        return builds

    def get_queue(self):
        """获取队列信息"""
        resp = self.session.get(f'{self.url}/queue/api/json')
        return resp.json()

    def create_job(self, job_name, config_xml):
        """创建 Job"""
        crumbs = {'Jenkins-Crumb': self.get_crumb()}
        headers = {**crumbs, 'Content-Type': 'application/xml'}
        resp = self.session.post(
            f'{self.url}/createItem?name={quote(job_name)}',
            headers=headers,
            data=config_xml
        )
        return resp.status_code == 200

    def delete_job(self, job_name):
        """删除 Job"""
        crumbs = {'Jenkins-Crumb': self.get_crumb()}
        resp = self.session.post(
            f'{self.url}/job/{quote(job_name)}/doDelete',
            headers=crumbs
        )
        return resp.status_code in [200, 302]


# 使用示例
if __name__ == '__main__':
    client = JenkinsClient(
        url='http://jenkins:8080',
        username='admin',
        token='your_api_token'
    )

    # 获取所有 Job
    jobs = client.get_jobs()
    for job in jobs:
        print(f"{job['name']}: {job['color']}")

    # 触发构建
    client.build_job('my-pipeline', {
        'BRANCH': 'main',
        'ENV': 'dev'
    })

    # 查看最近的构建详情
    builds = client.list_builds('my-pipeline', limit=5)
    for build in builds:
        print(f"Build #{build['number']}: {build['result']}")
```

# 6. 条件查询和 Webhook

## 6.1 等待构建完成

```python
import time

def wait_for_build(job_name, build_number, timeout=3600, interval=10):
    """等待构建完成，返回结果"""
    start_time = time.time()

    while time.time() - start_time < timeout:
        info = client.get_build_info(job_name, build_number)

        if not info.get('building', True):
            return info.get('result')

        print(f"Build #{build_number} still running...")
        time.sleep(interval)

    raise TimeoutError(f"Build #{build_number} did not complete in {timeout}s")


# 使用
build_number = 42
result = wait_for_build('my-pipeline', build_number)
print(f"Build completed with result: {result}")
```

## 6.2 Webhook 接收器

```python
# 外部 Webhook 触发 Jenkins 构建
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)
client = JenkinsClient('http://jenkins:8080', 'admin', 'token')
WEBHOOK_SECRET = b'shared-secret-key'

@app.route('/webhook/github', methods=['POST'])
def github_webhook():
    """接收 GitHub Webhook"""
    # 验证签名
    signature = request.headers.get('X-Hub-Signature-256', '')
    payload = request.get_data()
    expected = 'sha256=' + hmac.new(
        WEBHOOK_SECRET, payload, hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        return jsonify({'error': 'Invalid signature'}), 401

    event = request.headers.get('X-GitHub-Event')
    data = request.json

    if event == 'push':
        branch = data['ref'].split('/')[-1]
        repo = data['repository']['full_name']
        print(f"Push to {repo}: {branch}")

        # 触发对应 Job
        client.build_job(repo.replace('/', '-'), {
            'BRANCH': branch
        })

    return jsonify({'status': 'ok'}), 200


@app.route('/webhook/gitlab', methods=['POST'])
def gitlab_webhook():
    """接收 GitLab Webhook"""
    token = request.headers.get('X-Gitlab-Token', '')
    if token != 'gitlab-webhook-secret':
        return jsonify({'error': 'Invalid token'}), 401

    data = request.json
    event = request.headers.get('X-Gitlab-Event')

    if event == 'Push Hook':
        branch = data['ref'].split('/')[-1]
        project = data['project']['path_with_namespace']
        print(f"Push to {project}: {branch}")
        client.build_job(project.replace('/', '-'), {
            'BRANCH': branch
        })

    return jsonify({'status': 'ok'}), 200


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

# 7. 批量管理脚本

## 7.1 健康检查

```bash
#!/bin/bash
# jenkins-api-health.sh

JENKINS_URL="http://jenkins:8080"
AUTH="admin:token"

echo "=== Jenkins API Health Check ==="

# 1. API 可用性
echo -n "1. API Access: "
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -u $AUTH $JENKINS_URL/api/json)
if [ "$HTTP_CODE" == "200" ]; then
    echo "OK ($HTTP_CODE)"
else
    echo "FAIL ($HTTP_CODE)"
fi

# 2. Job 数量
echo -n "2. Total Jobs: "
JOBS=$(curl -s -u $AUTH "$JENKINS_URL/api/json?tree=jobs[name]" | jq '.jobs | length')
echo "$JOBS"

# 3. 最近失败的构建
echo "3. Recent Failed Builds: "
curl -s -u $AUTH "$JENKINS_URL/api/json?tree=jobs[name,lastFailedBuild[number,url]]" | \
    jq -r '.jobs[] | select(.lastFailedBuild != null) | "\(.name): #\(.lastFailedBuild.number)"'

# 4. 队列大小
echo -n "4. Queue Size: "
QUEUE=$(curl -s -u $AUTH $JENKINS_URL/queue/api/json | jq '.items | length')
echo "$QUEUE"

# 5. 节点状态
echo "5. Node Status: "
curl -s -u $AUTH "$JENKINS_URL/computer/api/json?tree=computer[displayName,offline,numExecutors,idle]" | \
    jq -r '.computer[] | "\(.displayName): offline=\(.offline) executors=\(.numExecutors) idle=\(.idle)"'
```

## 7.2 批量清理

```bash
#!/bin/bash
# jenkins-cleanup.sh

JENKINS_URL="http://jenkins:8080"
AUTH="admin:token"
DAYS_TO_KEEP=30

echo "=== Jenkins Cleanup ==="

# 1. 删除旧构建
echo "1. Deleting builds older than $DAYS_TO_KEEP days..."

curl -s -u $AUTH "$JENKINS_URL/api/json?tree=jobs[name,url]" | \
    jq -r '.jobs[].name' | while read job; do
        echo "  Processing $job..."

        # 获取构建列表
        builds=$(curl -s -u $AUTH "$JENKINS_URL/job/$job/api/json?tree=builds[number,timestamp]" | \
            jq -r --arg cutoff "$(date -d "$DAYS_TO_KEEP days ago" +%s)000" \
            '.builds[] | select(.timestamp < ($cutoff | tonumber)) | .number')

        # 删除旧构建
        for build in $builds; do
            echo "    Deleting build #$build..."
            curl -s -X POST -u $AUTH \
                "http://jenkins:8080/job/$job/$build/doDelete"
        done
    done

# 2. 清理工作空间
echo "2. Cleaning workspaces..."

curl -s -u $AUTH "$JENKINS_URL/api/json?tree=jobs[name,url,lastBuild[url]]" | \
    jq -r '.jobs[] | select(.lastBuild != null) | .name' | while read job; do
        echo "  Cleaning workspace for $job..."
        curl -X POST -u $AUTH \
            "http://jenkins:8080/job/$job/doWipeOutWorkspace"
    done

echo "=== Cleanup Complete ==="
```
