# Jenkins Pipeline Patterns for Data Engineering

> Patterns and techniques for building Jenkins CI/CD pipelines that deploy, execute, and monitor data engineering workloads. Examples drawn from GCP-based ETL orchestration projects.

**Related:** [[GitHub Actions for Python Projects]] | [[GitLab CI-CD for dbt]] | [[Docker & Container Patterns]]

---

## 1 - Pipeline Fundamentals

Jenkins supports two pipeline syntaxes. **Declarative** pipelines (wrapped in a `pipeline {}` block) are the standard for data engineering work because they enforce structure and make stages visible in the Blue Ocean UI.

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout')  { steps { checkout scm } }
        stage('Build')     { steps { sh './build.sh' } }
        stage('Test')      { steps { sh './run_tests.sh' } }
        stage('Deploy')    { steps { sh './deploy.sh' } }
    }

    post {
        always  { archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true }
        success { echo 'Pipeline completed successfully' }
        failure { echo 'Pipeline failed - check logs' }
    }
}
```

The `post` block runs after all stages. Use `always` for cleanup and artifact archiving, `success`/`failure` for conditional notifications.

---

## 2 - Parameterised Builds

Parameters turn a pipeline into a self-service tool. Data teams commonly need environment selectors, job pickers, and toggle flags.

```groovy
parameters {
    choice(
        name: 'environment',
        choices: ['dev', 'test', 'uat', 'prod'],
        description: 'Target environment for deployment'
    )
    choice(
        name: 'stack_action',
        choices: ['deploy-single-job', 'bulk-deploy', 'delete-job'],
        description: 'Deployment action to perform'
    )
    string(
        name: 'job_parameters',
        defaultValue: '{}',
        description: 'JSON parameters to pass to ETL job'
    )
    booleanParam(
        name: 'skip_tests',
        defaultValue: false,
        description: 'Skip running tests during deployment'
    )
    booleanParam(
        name: 'dry_run',
        defaultValue: false,
        description: 'Perform dry run without actual execution'
    )
}
```

Access parameters with `params.environment`, `params.dry_run`, etc. throughout the pipeline.

---

## 3 - Multi-Stage Pipeline Design

A well-structured data pipeline follows: **Init --> Validate --> Build --> Test --> Deploy --> Post-Analysis --> Notify**.

```groovy
stages {
    stage('Initialize') {
        steps {
            script {
                echo "Environment: ${params.environment}"
                echo "Build: ${BUILD_NUMBER}"
                validateCredentials()
            }
        }
    }
    stage('Clean Workspace & Checkout') {
        steps {
            cleanWs()
            checkout scm
        }
    }
    stage('Build Docker Image') {
        steps {
            sh """
                docker build -f docker/Dockerfile \
                    -t etl_deploy:${BUILD_NUMBER} \
                    -t etl_deploy:latest .
            """
        }
    }
    stage('Run Tests') {
        when { not { params.skip_tests } }
        steps { sh './scripts/run_pytest.sh' }
    }
    stage('Deploy') {
        steps { script { runDeployment() } }
    }
    stage('Post-execution Analysis') {
        steps {
            script {
                generateExecutionReport()
                runDataQualityChecks()
            }
        }
    }
}
```

Separate **deployment** pipelines (build, test, ship code) from **execution** pipelines (run ETL jobs, monitor, report). Use distinct Jenkinsfiles for each concern (e.g. `Jenkinsfile` and `Jenkinsfile.execution`).

---

## 4 - Shared Libraries

Place reusable Groovy functions in a `jenkins/shared/` directory. Each file returns `this` so it can be loaded as a library object.

```groovy
// jenkins/shared/gcp_utils.groovy
def authenticateGcp(String serviceAccountPath = null) {
    if (serviceAccountPath) {
        sh "gcloud auth activate-service-account --key-file=${serviceAccountPath}"
    }
    sh "gcloud auth application-default print-access-token > /dev/null"
    echo "GCP authentication successful"
}

def getProjectId() {
    return sh(script: "gcloud config get-value project", returnStdout: true).trim()
}

return this
```

Load shared libraries in a pipeline with `load`:

```groovy
stage('Initialize') {
    steps {
        script {
            def gcpUtils = load "jenkins/shared/gcp_utils.groovy"
            gcpUtils.authenticateGcp()

            def notifier = load "jenkins/shared/notification_utils.groovy"
            notifier.notifySuccess(environment: params.environment)
        }
    }
}
```

**Organisation tip:** Group utilities by concern -- `gcp_utils.groovy` for cloud operations, `notification_utils.groovy` for alerts, `etl_groups.groovy` for job configuration.

---

## 5 - GCP Operations in Groovy

### Authentication and project setup

```groovy
def setGcpEnvironment(String environment) {
    env.GCP_PROJECT_ID = sh(script: "gcloud config get-value project", returnStdout: true).trim()
    env.GCP_REGION = env.GCP_REGION ?: 'us-central1'
}
```

### GCS bucket management

```groovy
def bucketExists(String bucketName) {
    def result = sh(
        script: "gsutil ls gs://${bucketName} 2>/dev/null || echo 'NOT_FOUND'",
        returnStdout: true
    ).trim()
    return !result.contains('NOT_FOUND')
}

def createBucketIfNotExists(String bucketName, String location = 'US') {
    if (!bucketExists(bucketName)) {
        sh "gsutil mb -l ${location} gs://${bucketName}"
    }
}
```

### BigQuery operations

```groovy
def createDatasetIfNotExists(String datasetId, String location = 'US') {
    def project = sh(script: "gcloud config get-value project", returnStdout: true).trim()
    def result = sh(
        script: "bq show --dataset ${project}:${datasetId} 2>/dev/null || echo 'NOT_FOUND'",
        returnStdout: true
    ).trim()
    if (result.contains('NOT_FOUND')) {
        sh "bq mk --dataset --location=${location} ${project}:${datasetId}"
    }
}

def runBigQueryQuery(String query) {
    sh "bq query --use_legacy_sql=false '${query}'"
}
```

### Dataflow job monitoring

```groovy
def getDataflowJobStatus(String jobId, String region = 'us-central1') {
    return sh(
        script: "gcloud dataflow jobs describe ${jobId} --region=${region} --format='value(currentState)'",
        returnStdout: true
    ).trim()
}
```

### Secret Manager access

```groovy
def getSecret(String secretName, String version = 'latest') {
    return sh(
        script: "gcloud secrets versions access ${version} --secret=${secretName}",
        returnStdout: true
    ).trim()
}
```

### Quota checking

```groovy
def checkQuotas(String service = 'compute') {
    sh """
        gcloud ${service} project-info describe \
            --format='table(quotas[].metric,quotas[].usage,quotas[].limit)' | \
        awk 'NR>1 { if (\$2 > \$3 * 0.8) print "WARNING: " \$1 " at " \$2 "/" \$3 }'
    """
}
```

---

## 6 - Docker Integration

Build images inside the pipeline and use them as execution containers.

```groovy
environment {
    DOCKER_IMAGE = "etl_deploy"
    DOCKER_TAG = "${BUILD_NUMBER}"
}

stage('Build Docker Image') {
    steps {
        sh """
            docker build -f docker/Dockerfile \
                -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                -t ${DOCKER_IMAGE}:latest .
        """
    }
}

stage('Run in Container') {
    steps {
        sh """
            docker run --rm \
                -v ${WORKSPACE}:/workspace \
                -w /workspace \
                -e ENVIRONMENT=${params.environment} \
                ${DOCKER_IMAGE}:${DOCKER_TAG} \
                /workspace/scripts/deploy.sh
        """
    }
}
```

Always clean up in the `post` block:

```groovy
post {
    always {
        sh """
            docker ps -a -q --filter ancestor=${DOCKER_IMAGE}:${DOCKER_TAG} | xargs -r docker stop
            docker ps -a -q --filter ancestor=${DOCKER_IMAGE}:${DOCKER_TAG} | xargs -r docker rm
            docker system prune -f
        """
    }
}
```

See also: [[Docker & Container Patterns]]

---

## 7 - ETL Job Orchestration

### Configuration-driven job definitions

Use a Groovy config class to define ETL groups and individual scripts, enabling dynamic dropdown population:

```groovy
class ETLConfig {
    static def individualScripts = [
        'customer_order_frequency', 'sales_etl',
        'inventory_etl', 'product_analytics'
    ]
    static def etlGroups = [
        'daily_batch_group':   ['customer_order_frequency', 'sales_etl', 'inventory_etl'],
        'weekly_analytics':    ['product_analytics', 'customer_segmentation'],
        'all_etls':            individualScripts
    ]

    static def getAllOptions() {
        def options = ['']
        etlGroups.keySet().each { options.add("GROUP:${it}") }
        options.addAll(individualScripts)
        return options
    }

    static def getScripts(selection) {
        if (selection?.startsWith('GROUP:')) {
            return etlGroups[selection.substring(6)] ?: []
        }
        return selection ? [selection] : []
    }
}
return ETLConfig.getAllOptions()
```

### Parallel bulk execution

```groovy
stage('Execute Bulk ETL Jobs') {
    steps {
        script {
            def etlJobs = getAllEtlJobs()
            def parallelJobs = [:]
            etlJobs.each { job ->
                parallelJobs[job] = {
                    runSingleEtlJob(job, params.environment, params.job_parameters)
                }
            }
            parallel parallelJobs
        }
    }
}
```

### Job completion polling with timeout

```groovy
def waitForJobCompletion(String jobId) {
    timeout(time: 1, unit: 'HOURS') {
        def completed = false
        while (!completed) {
            def status = getDataflowJobStatus(jobId)
            if (status in ['JOB_STATE_DONE', 'JOB_STATE_UPDATED']) {
                completed = true
            } else if (status in ['JOB_STATE_FAILED', 'JOB_STATE_CANCELLED']) {
                error("Job failed with status: ${status}")
            } else {
                sleep(30)
            }
        }
    }
}
```

### Cloud Scheduler integration

```groovy
def createCloudSchedulerJob(Map config) {
    sh """
        gcloud scheduler jobs create http ${config.job_name}-${config.environment} \
            --schedule='${config.cron_expression}' \
            --uri='${JENKINS_URL}/job/${JOB_NAME}/buildWithParameters' \
            --http-method=POST \
            --message-body='execution_mode=run-single-etl&environment=${config.environment}&etl_job=${config.job_name}' \
            --time-zone='UTC'
    """
}
```

### Job cancellation with confirmation gate

```groovy
stage('Cancel Running Jobs') {
    when { expression { params.execution_mode == 'cancel-running-jobs' } }
    steps {
        script {
            input message: 'Are you sure you want to cancel running jobs?', ok: 'Yes, Cancel'
            sh """
                gcloud dataflow jobs list --region=${GCP_REGION} \
                    --filter='state=JOB_STATE_RUNNING AND name~${params.etl_job}' \
                    --format='value(JOB_ID)' | \
                while read job_id; do
                    gcloud dataflow jobs cancel \$job_id --region=${GCP_REGION}
                done
            """
        }
    }
}
```

---

## 8 - Notification Patterns

Build a reusable notification utility that dispatches to multiple channels (Slack, email, Teams) based on environment variables.

```groovy
def notifySuccess(Map config = [:]) {
    def payload = [
        status: 'success',
        title: 'ETL Pipeline Success',
        message: "Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} completed",
        environment: config.environment,
        duration: currentBuild.durationString,
        buildUrl: env.BUILD_URL
    ]
    if (env.SLACK_WEBHOOK_URL)  { sendSlackNotification(payload) }
    if (env.ETL_EMAIL_RECIPIENTS) { sendEmailNotification(payload) }
}

def notifyFailure(Map config = [:]) {
    def payload = [
        status: 'failure',
        title: 'ETL Pipeline Failure',
        message: "Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed",
        errorMessage: currentBuild.description ?: 'Check logs for details'
    ]
    if (env.SLACK_WEBHOOK_URL)  { sendSlackNotification(payload) }
    if (env.ETL_EMAIL_RECIPIENTS) { sendEmailNotification(payload) }
}
```

Wire notifications into `post` blocks:

```groovy
post {
    success { script { if (params.send_notifications) { notifySuccess() } } }
    failure {
        script {
            if (params.send_notifications) { notifyFailure() }
            if (env.CURRENT_JOB_ID) { cancelJob(env.CURRENT_JOB_ID) }
        }
    }
}
```

Use colour coding for status: `#36a64f` (success), `#ff0000` (failure), `#ffaa00` (warning).

---

## 9 - Conditional Execution

### `when` blocks for stage gating

```groovy
stage('Run Tests') {
    when { not { params.skip_tests } }
    steps { sh './run_tests.sh' }
}

stage('Deploy') {
    when {
        anyOf {
            expression { params.stack_action == 'deploy-single-job' }
            expression { params.stack_action == 'bulk-deploy' }
        }
    }
    steps { script { runDeployment() } }
}
```

### Dry-run support

```groovy
def runEtlJob(String jobName, Boolean dryRun) {
    def command = "gcloud dataflow jobs run ${jobName} --region=${GCP_REGION}"
    if (dryRun) {
        echo "DRY RUN - Would execute: ${command}"
        return "dry-run-${jobName}"
    } else {
        sh command
        return jobName
    }
}
```

### Environment gates

Combine `when` with `input` for production safeguards:

```groovy
stage('Production Deploy') {
    when { expression { params.environment == 'prod' } }
    steps {
        input message: 'Approve production deployment?', ok: 'Deploy'
        script { runDeployment() }
    }
}
```

---

## 10 - Best Practices

| Concern | Pattern |
|---|---|
| **Credential management** | Use `withCredentials {}` or GCP service account keys; never hardcode secrets |
| **Workspace cleanup** | Call `cleanWs()` at the start of each build to avoid stale state |
| **Timeout handling** | Wrap long-running stages in `timeout(time: 1, unit: 'HOURS')` |
| **Retry logic** | Use `retry(3) { sh '...' }` for flaky network calls |
| **Artifact archiving** | Always archive logs: `archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true` |
| **Docker cleanup** | Prune containers and images in `post { always {} }` |
| **Validation** | Validate parameters and prerequisites (quota, credentials) before execution stages |
| **Separate concerns** | Use distinct Jenkinsfiles: one for deployment, one for execution |
| **Dynamic config** | Drive dropdowns from Groovy config classes loaded with `load` |
| **Emergency cleanup** | On failure, cancel any in-flight cloud jobs to avoid resource waste |

---

**Tags:** #jenkins #cicd #data-engineering #groovy #gcp #pipelines
