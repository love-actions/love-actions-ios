# love-actions-ios

## About

Github Action for building & deploying iOS `.ipa` package of a [LÖVE](https://love2d.org/) framework based game to the **App Store**.

### Related actions

See related actions below:

[Love actions bare package](https://github.com/marketplace/actions/love-actions-bare-package)

[Love actions for testing](https://github.com/marketplace/actions/love-actions-for-testing)

[Love actions for android](https://github.com/marketplace/actions/love-actions-for-android)

[Love actions for Linux](https://github.com/marketplace/actions/love-actions-for-linux)

[Love actions for macOS AppStore](https://github.com/marketplace/actions/love-actions-for-macos-appstore)

[Love actions for macOS portable](https://github.com/marketplace/actions/love-actions-for-macos-portable)

[Love actions for Windows](https://github.com/marketplace/actions/love-actions-for-windows)

## Quick example

```yaml
- name: Build macOS packages
  id: build-packages
  uses: love-action/love-actions-macos-appstore@v1
  with:
    app-name: My Love Game
    bundle-id: org.love2d.my-game
    copyright: Copyright © 2020-2022 XXX Co. All Rights Reserved.
    icon-path: ./assets/macOS/icon.icns
    love-ref: "11.4"
    love-patch: "./love.patch"
    love-package: ./game.love
    libs-path: ./libs
    extra-assets: ./README.md ./license.txt
    product-name: my_game
    version-string: 2.3.4
    output-folder: ./dist
    apple-development-base64: ${{ secrets.APPLE_CERT_APPLE_DEVELOPMENT_BASE64 }}
    apple-development-password: ${{ secrets.APPLE_CERT_APPLE_DEVELOPMENT_PWD }}
    api-key: ${{ secrets.APPLE_API_KEY }}
    api-key-id: ${{ secrets.APPLE_API_KEY_ID }}
    api-issuer-id: ${{ secrets.APPLE_API_ISSUER_ID }}
    team-id: ${{ secrets.APPLE_DEVELOPER_TEAM_ID }}
    apple-id: ${{ secrets.APPLE_APPLE_ID }}
```

## With [Love actions bare package](https://github.com/marketplace/actions/love-actions-bare-package) and [Love actions for testing](https://github.com/marketplace/actions/love-actions-for-testing)

```yml
env:
  BUILD_TYPE: ${{ fromJSON('["dev", "release"]')[startsWith(github.ref, 'refs/tags/v')] }}
  CORE_LOVE_PACKAGE_PATH: ./core.love
  CORE_LOVE_ARTIFACT_NAME: core_love_package
  PRODUCT_NAME: my_love_app
  BUNDLE_ID: com.example.myloveapp

jobs:
  build-core:
    runs-on: ubuntu-latest
    env:
      OUTPUT_FOLDER: ./build
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive
      - name: Build core love package
        uses: love-actions/love-actions-core@v1
        with:
          build-list: ./media/ ./parts/ ./Zframework/ ./conf.lua ./main.lua ./version.lua
          package-path: ${{ env.CORE_LOVE_PACKAGE_PATH }}
      - name: Upload core love package
        uses: actions/upload-artifact@v3
        with:
          name: ${{ env.CORE_LOVE_ARTIFACT_NAME }}
          path: ${{ env.CORE_LOVE_PACKAGE_PATH }}
  auto-test:
    runs-on: ubuntu-latest
    needs: build-core
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive
      - name: Love actions for testing
        uses: love-actions/love-actions-test@v1
        with:
          font-path: ./parts/fonts/proportional.otf
          language-folder: ./parts/language
  build-ios:
    runs-on: macos-latest
    needs: [build-core, auto-test]
    env:
      OUTPUT_FOLDER: ./build
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive
      # Download your core love package here
      - name: Download core love package
        uses: actions/download-artifact@v3
        with:
          name: ${{ env.CORE_LOVE_ARTIFACT_NAME }}
      # This is an example dynamic library
      - name: Download ColdClear
        uses: ./.github/actions/get-cc
        with:
          platform: iOS
          dir: ./ColdClear
      - name: Build iOS packages
        id: build-packages
        uses: love-actions/love-actions-ios@v1
        with:
          app-name: ${{ env.PRODUCT_NAME }}
          bundle-id: ${{ env.BUNDLE_ID }}
          copyright: "Copyright © 2019-2022 26F-Studio. Some Rights Reserved."
          icon-path: ./.github/build/iOS/${{ env.BUILD_TYPE }}/icon
          love-patch: ./.github/build/iOS/love.patch
          love-package: ${{ env.CORE_LOVE_PACKAGE_PATH }}
          libs-path: ./ColdClear/arm64/
          product-name: ${{ env.PRODUCT_NAME }}
          version-string: "1.0.0"
          output-folder: ${{ env.OUTPUT_FOLDER }}
          apple-development-base64: ${{ secrets.APPLE_CERT_APPLE_DEVELOPMENT_BASE64 }}
          apple-development-password: ${{ secrets.APPLE_CERT_APPLE_DEVELOPMENT_PWD }}
          api-key: ${{ secrets.APPLE_API_KEY }}
          api-key-id: ${{ secrets.APPLE_API_KEY_ID }}
          api-issuer-id: ${{ secrets.APPLE_API_ISSUER_ID }}
          team-id: ${{ secrets.APPLE_DEVELOPER_TEAM_ID }}
          apple-id: ${{ secrets.APPLE_APPLE_ID }}
          external-test: ${{ startsWith(github.ref, 'refs/tags/pre') }}
          store-release: ${{ startsWith(github.ref, 'refs/tags/v') }}
      - name: Upload logs artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ needs.get-info.outputs.base-name }}_iOS_logs
          path: |
            ${{ env.OUTPUT_FOLDER }}/DistributionSummary.plist
            ${{ env.OUTPUT_FOLDER }}/ExportOptions.plist
            ${{ env.OUTPUT_FOLDER }}/Packaging.log
      - name: Upload ipa artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ needs.get-info.outputs.base-name }}_iOS_ipa
          path: ${{ env.OUTPUT_FOLDER }}/${{ steps.process-app-name.outputs.product-name }}.ipa
```
## All inputs

| Name                           | Required  | Default                                        | Description                                                                                                                                     |
| :----------------------------- | --------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `app-name`                   | `false` | `"LÖVE for macOS"`                          | App display name. Used in `platform/xcode/ios/love-ios.plist`                                                                                 |
| `bundle-id`                  | `false` | `"org.love2d.macOS"`                         | App bundle id. Used in `platform/xcode/love.xcodeproj/project.pbxproj`                                                                        |
| `copyright`                  | `false` | `""`                                         | App copyright info. Used in `platform/xcode/ios/love-ios.plist`                                                                               |
| `icon-path`                  | `false` | `"./icon.icns"`                              | `.icns` format icon's path. Used in `platform/xcode/Images.xcassets/OS X AppIcon.appiconset`                                                |
| `love-ref`                   | `false` | `"c35356c841976eb6f370347b81eec845d5520338"` | LÖVE git ref. Could be commit hash, tags or branch name                                                                                        |
| `love-patch`                 | `false` | `""`                                         | Git patch file path for the LÖVE repo. The patch must start from `love-ref`. You can use `git diff -p <tag1> <tag2>` to get the patch file |
| `love-package`               | `false` | `"./game.love"`                              | `.love` game package file path                                                                                                                |
| `libs-path`                  | `false` | `""`                                         | Path to the*static* libraries folder. Would copy all contents to `platform/xcode/` excluding top folder                                     |
| `extra-assets`               | `false` | `""`                                         | List of folder & file paths to be added to `platform/xcode/`. Separated by spaces                                                             |
| `product-name`               | `false` | `"love_app"`                                 | Base name of the package. Used to rename products                                                                                               |
| `version-string`             | `false` | `"11.4"`                                     | App version string no more than 3 numbers. Used in `platform/xcode/love.xcodeproj/project.pbxproj`                                            |
| `output-folder`              | `false` | `"./build"`                                  | Built packages output folder                                                                                                                    |
| `apple-development-base64`   | `true`  | `""`                                         | Apple Development certificate base64 content. Used to sign the app                                                                              |
| `apple-development-password` | `true`  | `""`                                         | Apple Development certificate password. Used to sign the app                                                                                    |
| `api-key`                    | `true`  | `""`                                         | App Store Connect API key content. Used to automaticly update profiles, app IDs and certificates                                                |
| `api-key-id`                 | `true`  | `""`                                         | App Store Connect API key ID. Used to automaticly update profiles, app IDs and certificates                                                     |
| `api-issuer-id`              | `true`  | `""`                                         | App Store Connect API issuer ID. Used to automaticly update profiles, app IDs and certificates                                                  |
| `team-id`                    | `true`  | `""`                                         | Developer team id. Used to sign the app                                                                                                         |
| `apple-id`                   | `true`  | `""`                                         | App Apple ID. Used to upload the package                                                                                                        |

## All outputs

| Name              | Example                 | Description                                                                                     |
| :---------------- | ----------------------- | ----------------------------------------------------------------------------------------------- |
| `package-paths` | `./build/my_game.ipa` | Built packages' paths in a bash-style list relative to the repository root, separated by spaces |
