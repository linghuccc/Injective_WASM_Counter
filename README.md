## 1. 准备工作

### 1.1 首先安装Rust:

确认系统是否已安装Rust:

```sh
rustc --version
cargo --version
```

如果没有，请通过下列命令安装：

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 将Rust加载到当前shell会话中
source "$HOME/.cargo/env"
```

### 1.2 其次安装`wasm32-unknown-unknown` target:

确认是否已安装wasm32-unknown-unknown:

```sh
rustup target list --installed
# 或使用
rustup show
```

'installed targets'下应该显示'wasm32-unknown-unknown'，如果没有，请通过下列命令安装：

```sh
rustup target add wasm32-unknown-unknown
```


### 1.3 接着安装[cargo-generate](https://github.com/ashleygwilliams/cargo-generate)和cargo-run-script：

确认是否已安装'cargo generate'和'cargo run-script':

```sh
cargo --list
```

命令列表中应显示'generate'和'run-script'，如果没有，请通过下列命令安装：

```sh
cargo install cargo-generate --features vendored-openssl

cargo install cargo-run-script
```

### 1.4 然后安装Injectived：

确认是否已安装injectived:

```sh
injectived version
```

如果没有，请通过下列命令安装：

```sh
wget https://github.com/InjectiveLabs/injective-chain-releases/releases/download/v1.15.0-1748457819/linux-amd64.zip

unzip linux-amd64.zip

sudo mv injectived peggo /usr/bin
sudo mv libwasmvm.x86_64.so /usr/lib
```

### 1.5 最后安装yj：

确认是否已安装yj:

```sh
yj --version
```

如果没有，请通过下列命令安装：

```sh
sudo curl -L https://github.com/sclevine/yj/releases/latest/download/yj-linux-amd64 -o /usr/local/bin/yj

sudo chmod +x /usr/local/bin/yj
```

## 2. 创建Counter项目

### 2.1 创建项目

创建Counter项目,并转到它的文件夹:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --name counter
cd counter/
```

### 2.2 编译、运行测试和生成JSON模式

现在你已经创建了Counter合约，确保在进行任何更改之前可以编译和运行它。
进入仓库并执行:

```sh
# 这将在./target/wasm32-unknown-unknown/release/YOUR_NAME_HERE.wasm中生成wasm构建
cargo wasm

# 这会运行单元测试并提供有用的回溯
RUST_BACKTRACE=1 cargo unit-test

# 自动生成json模式
cargo schema
```

### 2.3 为生产环境准备Wasm字节码

运行`rust-optimizer`:

```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer:0.16.0
```

或者，如果你在arm64机器上，你应该使用为arm64构建的docker镜像：

```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/optimizer-arm64:0.16.0
```

这会生成一个`artifacts`目录，其中包含`PROJECT_NAME.wasm`，以及`checksums.txt`，包含wasm文件的Sha256哈希值。

## 3. 上传WSAM合约

### 3.1 设置RPC端口

```sh
injectived config set client chain-id injective-888
injectived config set client node https://k8s.testnet.tm.injective.network:443
```

通过检查账户状态验证RPC端口:

```sh
injectived q bank balances ACCOUNT_ADDRESS
```

确保账户余额能正确显示。

### 3.2 添加钱包账户:

```sh
# 生成一个新账户
injectived keys add ACCOUNT_NAME
# 或使用现有账户
injectived keys unsafe-import-eth-key ACCOUNT_NAME $PRIVATE_KEY
```

验证账户:

```sh
injectived keys list
```

确保账户内有INJ测试币:

```sh
injectived q bank balances ACCOUNT_ADDRESS
```

### 3.3 上传WSAM合约:

```sh
# 在"injective-core-staging"容器内，或如果本地运行injectived，则在合约目录中
TXHASH=$(
  yes $KEYRING_PASSWORD | injectived tx wasm store artifacts/counter.wasm \
  --from=$(echo $ACCOUNT_ADDRESS) \
  --chain-id="injective-888" \
  --yes --fees=1000000000000000inj --gas=2000000 \
  --node=https://testnet.sentry.tm.injective.network:443 \
  | yj -yj | jq -r .txhash
)
echo "txhash: $TXHASH"

# 得到Code ID
CODE_ID=$(
  injectived query tx $TXHASH --node=https://testnet.sentry.tm.injective.network:443 \
  | yj -yj | jq -r '.events[] | select(.type == "store_code") | .attributes[] | select(.key == "code_id") | .value'
)
echo "code_id: $CODE_ID"
```

## 4. 与合约交互:

### 4.1 实例化合约:

```sh
INIT='{"count":99}'
echo "instantiate: ${INIT}"
TXHASH=$(
  yes $KEYRING_PASSWORD | injectived tx wasm instantiate $CODE_ID $INIT \
  --label="CounterTestInstance" \
  --from=$(echo $ACCOUNT_ADDRESS) \
  --chain-id="injective-888" \
  --yes --fees=1000000000000000inj \
  --gas=2000000 \
  --no-admin \
  --node=https://testnet.sentry.tm.injective.network:443 \
  | yj -yj | jq -r .txhash
)
echo "txhash: $TXHASH"

# 得到合约地址
CONTRACT=$(
  injectived query tx $TXHASH --node=https://testnet.sentry.tm.injective.network:443 \
  | yj -yj | jq .events[] -c | jq .attributes[] -c \
  | grep contract_address | jq -r .value | head -n1
)
echo "contract: $CONTRACT"
```

### 4.2 检查合约信息:

```sh
injectived query wasm contract $CONTRACT --node=https://testnet.sentry.tm.injective.network:443
```

### 4.3 获取合约计数:

```sh
GET_COUNT_QUERY='{"get_count":{}}'
echo "query: ${GET_COUNT_QUERY}"
injectived query wasm contract-state smart $CONTRACT "$GET_COUNT_QUERY" \
--node=https://testnet.sentry.tm.injective.network:443 \
--output json | jq .
```

### 4.4 执行Increment程序:

```sh
INCREMENT='{"increment":{}}'
echo "execute: ${INCREMENT}"
TXHASH=$(
  yes $KEYRING_PASSWORD | injectived tx wasm execute $CONTRACT "$INCREMENT" --from=$(echo $ACCOUNT_ADDRESS) \
  --chain-id="injective-888" \
  --yes --fees=1000000000000000inj --gas=2000000 \
  --node=https://testnet.sentry.tm.injective.network:443 \
  --output json | jq -r .txhash
)
echo "txhash: $TXHASH"
```

再次获取合约计数以验证:

```sh
GET_COUNT_QUERY='{"get_count":{}}'
echo "query: ${GET_COUNT_QUERY}"
injectived query wasm contract-state smart $CONTRACT "$GET_COUNT_QUERY" \
--node=https://testnet.sentry.tm.injective.network:443 \
--output json | jq .
```