---
title: "LocalStackでAirflowのEcsRunTaskOperatorを使う時に発生するKeyError対処法"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["airflow", "localstack", "ecs", "aws", "docker"]
published: false
---

# はじめに

中野製菓のかりんとうが大好きです。

AirflowのEcsRunTaskOperatorをLocalStackでテストしようとした際に、`KeyError: 'failures'`エラーが発生することがあります。この記事では、その原因と解決方法について詳しく解説します。

## TL;DR

LocalStackでAirflowの`EcsRunTaskOperator`を使用するには、カスタムオペレーターの実装が必要です。これは、LocalStackとAWS ECSのAPIレスポンス形式の違いが原因です。

## 問題の概要

LocalStack環境でEcsRunTaskOperatorを実行すると、以下のようなエラーが発生します：

```python
KeyError: 'failures'
```

### エラーログの例

```log
[2025-05-13, 09:58:33 JST] {ecs.py:636} INFO - Running ECS Task - Task definition: MY_TASK_DEFINITION - on cluster MY_CLUSTER
[2025-05-13, 09:58:33 JST] {taskinstance.py:3310} ERROR - Task failed with exception
Traceback (most recent call last):
  ...
  File "/home/airflow/.local/lib/python3.12/site-packages/airflow/providers/amazon/aws/operators/ecs.py", line 636, in _start_task
    failures = response["failures"]
               ~~~~~~~~^^^^^^^^^^^^
KeyError: 'failures'
```

## 原因の詳細

この問題の根本原因は、AWS ECSとLocalStackの`run-task`APIレスポンス形式の違いにあります。

### AWS ECSの場合

[AWS公式ドキュメント](https://docs.aws.amazon.com/cli/latest/reference/ecs/run-task.html#output)によると、`run-task`のレスポンスには`failures`プロパティが含まれています：

```json
{
    "tasks": [
        {
            "attachments": [],
            "attributes": [
                {
                    "name": "ecs.cpu-architecture",
                    "value": "x86_64"
                }
            ],
            "availabilityZone": "us-east-1a",
            "ephemeralStorage": {
                "sizeInGiB": 20
            }
        }
    ],
    "failures": []
}
```

### LocalStackの場合

しかし、LocalStackの`awslocal ecs run-task`コマンドの出力には`failures`プロパティが含まれていません：

```json
{
    "tasks": [
        {
            "attachments": [],
            "attributes": [
                {
                    "name": "ecs.cpu-architecture",
                    "value": "x86_64"
                }
            ],
            "availabilityZone": "us-east-1a",
            "ephemeralStorage": {
                "sizeInGiB": 20
            }
        }
    ]
}
```

この違いにより、AirflowのEcsRunTaskOperatorが`response["failures"]`にアクセスしようとした際にKeyErrorが発生します。

## 解決方法

この問題を解決するために、LocalStack対応のカスタムオペレーターを作成します。

### カスタムオペレーターの実装

```python
from airflow.providers.amazon.aws.operators.ecs import EcsRunTaskOperator
from airflow.utils.decorators import apply_defaults


class CustomEcsRunTaskOperator(EcsRunTaskOperator):
    """
    LocalStack互換性のためのカスタムEcsRunTaskOperator
    レスポンスに'failures'キーが存在しない場合に対処する
    """

    @apply_defaults
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

    def _start_task(self):
        """
        LocalStackのrun_task API レスポンスに対応するため_start_taskをオーバーライド
        親クラスのrun_task呼び出しロジックと同様だが、'failures'の処理をより堅牢にする
        """
        self.log.info(
            "LocalStack互換性のためCustomEcsRunTaskOperatorの_start_taskを実行"
        )

        # run_taskのパラメータを構築
        api_params = {
            "taskDefinition": getattr(self, "task_definition", None),
            "cluster": getattr(self, "cluster", None),
            "count": getattr(self, "count", None),
            "enableECSManagedTags": getattr(self, "enable_ecs_managed_tags", None),
            "enableExecuteCommand": getattr(self, "enable_execute_command", None),
            "launchType": getattr(self, "launch_type", None),
            "networkConfiguration": getattr(self, "network_configuration", None),
            "overrides": getattr(self, "overrides", None),
            "placementConstraints": getattr(self, "placement_constraints", None),
            "placementStrategy": getattr(self, "placement_strategy", None),
            "platformVersion": getattr(self, "platform_version", None),
            "propagateTags": getattr(self, "propagate_tags", None),
            "startedBy": getattr(self, "started_by", None),
            "tags": getattr(self, "tags", None),
            "volumeConfigurations": getattr(self, "volume_configurations", None),
            "capacityProviderStrategy": getattr(
                self, "capacity_provider_strategy", None
            ),
        }

        if hasattr(self, "group") and self.group is not None:
            api_params["group"] = self.group

        run_task_params = {k: v for k, v in api_params.items() if v is not None}

        # 必須パラメータの検証
        if not run_task_params.get("taskDefinition"):
            raise ValueError(
                "taskDefinitionは必須パラメータですが、提供されていないかNoneです。"
            )
        
        if not run_task_params.get("cluster"):
            self.log.warning(
                "パラメータ'cluster'が設定されていません。デフォルトクラスターが使用されます。"
            )
        
        if not run_task_params.get("launchType") and not run_task_params.get(
            "capacityProviderStrategy"
        ):
            self.log.warning(
                "'launchType'も'capacityProviderStrategy'も設定されていません。"
                "クラスターのデフォルト容量プロバイダー戦略が使用されます。"
            )

        self.log.info(f"ECS run_taskを以下のパラメータで呼び出し: {run_task_params}")
        response = self.client.run_task(**run_task_params)

        # LocalStack対応: failuresキーが存在しない場合に対処
        failures = response.get("failures", [])

        if failures or not response.get("tasks"):
            self.log.error(
                f"ECSタスクの起動に失敗したか、タスクが報告されませんでした。"
                f"失敗: {failures}, 完全なレスポンス: {response}"
            )
            if failures:
                raise RuntimeError(
                    f"ECSタスクが失敗しました: {failures}. 完全なレスポンス: {response}"
                )
            if not response.get("tasks"):
                raise RuntimeError(
                    f"ECSタスクの起動でタスクが報告されませんでした。レスポンス: {response}"
                )

        task_arn = response["tasks"][0]["taskArn"]
        self.arn = task_arn
        self.log.info("ECSタスクが正常に開始されました: %s", task_arn)

        return task_arn
```

### カスタムオペレーターの使用方法

環境に応じてオペレーターを切り替える実装例：

```python
import os
from airflow.providers.amazon.aws.operators.ecs import EcsRunTaskOperator

# 環境変数でローカル環境かどうかを判定
is_local_env = os.getenv("IS_LOCAL")

if is_local_env:
    EcsOperator = CustomEcsRunTaskOperator
    logger.info("ローカル環境用のCustomEcsRunTaskOperatorを使用")
else:
    EcsOperator = EcsRunTaskOperator
    logger.info("本番環境用のEcsRunTaskOperatorを使用")

# タスクの定義
task = EcsOperator(
    task_id="sample_ecs_task",
    aws_conn_id="aws_conn",
    task_definition="my_task_definition",
    cluster="my_cluster",
    # その他のパラメータ...
)
```

## 注意事項

⚠️ **重要**: このカスタムオペレーターは本番環境では使用しないことを強く推奨します。これはローカル開発・テスト環境専用の回避策として使用してください。

## まとめ

LocalStackとAWS ECSのAPIレスポンス形式の違いにより、AirflowのEcsRunTaskOperatorでKeyErrorが発生する問題について解説しました。

### 要点

- **原因**: LocalStackの`run-task`レスポンスに`failures`プロパティが含まれていない
- **解決方法**: `response.get("failures", [])`を使用してフェイルセーフなカスタムオペレーターを実装
- **注意点**: 本番環境では標準のEcsRunTaskOperatorを使用すること

この解決方法により、LocalStack環境でもAirflowのECSタスクを正常に実行できるようになります。ローカル開発環境でのテストが効率的に行えるようになり、開発生産性の向上に貢献します。

## 参考資料

- [AWS ECS run-task API ドキュメント](https://docs.aws.amazon.com/cli/latest/reference/ecs/run-task.html#output)
- [Airflow EcsRunTaskOperator ドキュメント](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/operators/ecs.html)
- [LocalStack 公式ドキュメント](https://docs.localstack.cloud/)


