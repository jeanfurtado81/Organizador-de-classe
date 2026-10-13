name: Build APK do Diario de Classe

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      ANDROID_SDK_ROOT: ${{ runner.temp }}/android-sdk

    steps:
      - name: Baixar os arquivos do repositorio
        uses: actions/checkout@v4

      - name: Preparar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Preparar Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Instalar Android SDK
        run: |
          sudo apt-get update
          sudo apt-get install -y unzip wget
          mkdir -p "$ANDROID_SDK_ROOT/cmdline-tools"
          wget -q https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip -O /tmp/cmdline-tools.zip
          unzip -q /tmp/cmdline-tools.zip -d "$ANDROID_SDK_ROOT/cmdline-tools"
          yes | "$ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools/bin/sdkmanager" --sdk_root="$ANDROID_SDK_ROOT" "platform-tools" "platforms;android-31" "build-tools;31.0.0"
          echo "ANDROID_SDK_ROOT=$ANDROID_SDK_ROOT" >> "$GITHUB_ENV"
          echo "$ANDROID_SDK_ROOT/platform-tools" >> "$GITHUB_PATH"
          echo "$ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools/bin" >> "$GITHUB_PATH"

      - name: Extrair o projeto
        run: unzip -o diario-de-classe-android-offline.zip -d projeto

      - name: Preparar Android / Capacitor
        run: |
          cd projeto/diario-de-classe-android-offline
          npm ci || npm install
          npm install @capacitor/android@6.2.2 --no-fund --no-audit
          npx cap add android || true
          npx cap sync android

      - name: Criar keystore (se o secret existir)
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
        run: |
          if [ -n "$KEYSTORE_BASE64" ]; then
            printf '%s' "$KEYSTORE_BASE64" | base64 --decode > projeto/diario-de-classe-android-offline/android/keystore.jks
          fi

      - name: Gerar APK
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          cd projeto/diario-de-classe-android-offline/android
          chmod +x gradlew

          if [ -n "$KEYSTORE_PASSWORD" ] && [ -n "$KEY_ALIAS" ] && [ -n "$KEY_PASSWORD" ] && [ -f "keystore.jks" ]; then
            ./gradlew assembleRelease \
              -Pandroid.injected.signing.store.file="$PWD/keystore.jks" \
              -Pandroid.injected.signing.store.password="$KEYSTORE_PASSWORD" \
              -Pandroid.injected.signing.key.alias="$KEY_ALIAS" \
              -Pandroid.injected.signing.key.password="$KEY_PASSWORD" \
              --no-daemon
          else
            ./gradlew assembleDebug --no-daemon
          fi

      - name: Disponibilizar o APK para baixar
        uses: actions/upload-artifact@v4
        with:
          name: diario-de-classe-apk
          path: |
            projeto/diario-de-classe-android-offline/android/app/build/outputs/apk/**/*.apk
            projeto/diario-de-classe-android-offline/android/app/build/outputs/bundle/**/*.aab
          if-no-files-found: error
          retention-days: 7
