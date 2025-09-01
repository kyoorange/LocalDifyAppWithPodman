# Ollama + Dify セットアップガイド (Podman環境)

このガイドでは、Podman環境でDifyとOllamaを連携させて、ローカルでqwen3:0.6bモデルを使用する方法を説明します。

## 🚀 クイックスタート

### 1. Ollamaサービスの起動

```bash
# Ollamaサービスを起動（qwen3:0.6bモデルも自動ダウンロード）
./setup-ollama.sh
```

### 2. 手動で起動する場合

```bash
# Ollamaサービスを起動
podman-compose --profile ollama up -d ollama

# モデルをダウンロード
podman exec -it dify-docker-ollama-1 ollama pull qwen3:0.6b
```

## 🔧 Difyでの設定

### 1. Dify管理画面にアクセス
- ブラウザでDifyの管理画面を開く
- 通常は `http://localhost:8080` または設定したポート

### 2. LLMプロバイダーの追加（必須）

#### 基本設定
1. **設定** → **LLMプロバイダー** → **プロバイダー追加**
2. 以下の設定を入力：
   - **プロバイダー名**: `Ollama Local`
   - **プロバイダータイプ**: `Ollama`
   - **ベースURL**: `http://localhost:11434`
   - **モデル名**: `qwen3:0.6b`
   - **APIキー**: 空欄のまま（ローカル環境では不要）

#### 詳細設定
- **コンテキスト長**: `8192`（モデルの最大入力長）
- **最大トークン**: `4096`（生成される最大トークン数）
- **温度**: `0.7`（創造性の調整、0.0-1.0）
- **Top P**: `0.9`（核サンプリング、0.0-1.0）

### 3. アプリケーションでの使用

#### チャットボットアプリの作成
1. **アプリケーション作成** → **チャットボット**
2. **LLM設定**で、追加したOllamaプロバイダーを選択
3. モデル名として `qwen3:0.6b` を指定
4. **プロンプト設定**で適切なプロンプトを記述

#### ワークフローアプリの作成
1. **アプリケーション作成** → **ワークフロー**
2. **LLMノード**を追加
3. **LLM設定**で、追加したOllamaプロバイダーを選択
4. モデル名として `qwen3:0.6b` を指定

### 4. 高度な設定

#### カスタムプロンプトの設定
```yaml
# システムプロンプト例
system_prompt: |
  あなたは親切で知識豊富なAIアシスタントです。
  日本語で丁寧に回答してください。
  回答は簡潔で分かりやすくしてください。

# ユーザープロンプト例
user_prompt: |
  {{input}}
  
  上記の内容について、以下の点を考慮して回答してください：
  1. 要点を整理する
  2. 具体的な例を示す
  3. 実用的なアドバイスを提供する
```

#### モデルパラメータの調整
- **Temperature**: 低い値（0.1-0.3）で一貫性のある回答、高い値（0.7-1.0）で創造的な回答
- **Top P**: 低い値で確実性重視、高い値で多様性重視
- **Frequency Penalty**: 繰り返しを減らす（-2.0 から 2.0）
- **Presence Penalty**: 新しいトピックを奨励（-2.0 から 2.0）

## 📊 動作確認

### Ollamaサービスの状態確認
```bash
# サービス状態確認
podman-compose ps ollama

# ログ確認
podman-compose logs ollama

# 利用可能なモデル確認
podman exec -it dify-docker-ollama-1 ollama list
```

### APIテスト
```bash
# 簡単なテスト
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3:0.6b",
    "prompt": "こんにちは、簡単な自己紹介をお願いします。",
    "stream": false
  }'
```

### Difyでの動作確認
1. **テストチャット**で簡単な質問を送信
2. **ログ**でエラーがないか確認
3. **レスポンス時間**を確認（初回は遅い場合があります）

## 🛠️ トラブルシューティング

### よくある問題

1. **ポート11434が使用できない**
   ```bash
   # ポートの使用状況確認
   netstat -tlnp | grep 11434
   
   # 必要に応じてポート変更（docker-compose.yamlで設定）
   ```

2. **モデルのダウンロードが遅い**
   - インターネット接続を確認
   - 必要に応じてプロキシ設定を調整

3. **Difyで接続できない**
   - Ollamaサービスの起動確認
   - ファイアウォール設定の確認
   - ネットワーク設定の確認

4. **LLMプロバイダーが表示されない**
   - プロバイダータイプが正しく「Ollama」になっているか確認
   - ベースURLが正しいか確認
   - Difyの再起動を試す

5. **モデルが見つからないエラー**
   - モデル名が正確に一致しているか確認
   - `podman exec -it dify-docker-ollama-1 ollama list` で利用可能なモデルを確認

### ログの確認
```bash
# Ollamaサービスのログ
podman-compose logs -f ollama

# Dify APIサービスのログ
podman-compose logs -f api

# Dify Webサービスのログ
podman-compose logs -f web
```

## 🔄 モデルの更新・変更

### 新しいモデルの追加
```bash
# 例：llama3.1:8bモデルを追加
podman exec -it dify-docker-ollama-1 ollama pull llama3.1:8b

# 例：mistral:7bモデルを追加
podman exec -it dify-docker-ollama-1 ollama pull mistral:7b
```

### モデルの削除
```bash
# 不要なモデルを削除
podman exec -it dify-docker-ollama-1 ollama rm qwen3:0.6b
```

### モデルの更新
```bash
# モデルを最新版に更新
podman exec -it dify-docker-ollama-1 ollama pull qwen3:0.6b
```

## 📝 注意事項

- **メモリ使用量**: qwen3:0.6bは約1.2GBのメモリを使用します
- **初回起動**: モデルのダウンロードには時間がかかります
- **ネットワーク**: ローカル環境でのみ動作します
- **パフォーマンス**: GPUがない環境では処理が遅くなる可能性があります
- **コンテナエンジン**: Podman環境でのみ動作します
- **モデル名**: 正確なモデル名を使用する必要があります
- **設定変更**: プロバイダー設定を変更した後は、アプリケーションの再設定が必要な場合があります

## 🔧 Podman環境での起動方法

```bash
# Dify + Ollamaを同時起動
podman-compose --profile ollama up -d

# または個別起動
podman-compose up -d
podman-compose --profile ollama up -d ollama
```

## 🆘 サポート

問題が発生した場合は、以下を確認してください：

1. Podmanとpodman-composeのバージョン
2. システムリソース（メモリ、ディスク容量）
3. ネットワーク設定
4. ファイアウォール設定
5. Difyの設定（LLMプロバイダー、モデル名など）

詳細なログと共に問題を報告していただければ、より具体的な解決策を提供できます。
