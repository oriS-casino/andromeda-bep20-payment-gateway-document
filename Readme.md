# ANDROMEDA Bep20 Payment Gateway

Cung cấp kênh thanh toán trên mạng Binance Smart Chain ứng với các loại token Bep20.

## Mục lục

- [Library](#library)
- [Config](#config)
- [Deployment](#deployment)
- [Run Test](#run-test)
- [Database](#database)
    * [deposit_transactions](#deposit-transactions)
    * [withdraw_transactions](#withdraw-transactions)
    * [users](#users)
    * [last_watched_block](#last-watched-block)
    * [failed_request](#failed-request)
- [API](#api)
    * [Complete Player Deposit Transaction Redirect URL](#complete-player-deposit-transaction-redirect-url)
    * [Complete Transferred Tokens Redirect URL](#complete-transferred-tokens-redirect-url)
    * [Get User Address](#get-user-address)
    * [User Count](#user-count)
    * [Get User ID](#get-user-id)
    * [Get User Deposit Transactions Count](#get-user-deposit-transactions-count)
    * [Get User Completed Deposit Transactions](#get-user-completed-deposit-transactions)
    * [Get User Pending Deposit Transactions](#get-user-pending-deposit-transactions)
    * [Get User Withdraw Transactions Count](#get-user-withdraw-transactions-count)
    * [Get User Completed Withdraw Transactions](#get-user-completed-withdraw-transactions)
    * [Get User Pending Withdraw Transactions](#get-user-pending-withdraw-transactions)
    * [Withdraw](#withdraw)
    * [Model](#model)
        + [DepositTransaction](#deposittransaction)
        + [WithdrawTransaction](#withdrawtransaction)

<small><i><a href='https://github.com/dangnguyendota'>email: dangnguyendota@gmail.com</a></i></small>

## Library

Client viết bằng Golang: https://github.com/dangnguyendota-casino/go-andromeda-client

## Config

Trong thư mục `configs/` chứa file `config.example.yaml` là mẫu config, chứa các trường cần có.
Copy file `config.example.yaml` sang file `config.yaml` và sửa lại các trường này.

`config.example.yaml`

```yaml
log: ./logs
debug: false
maxWallets: 10000
amqpURI: "amqp://guest:guest@andromeda-rabbitmq:5672/"
mongoDNS: "mongodb://andromeda:andromeda@andromeda-mongodb:27017"
hotWalletMnemonic: ""
numberOfConfirmationsToApproveDepositTx: 15
tokens:
  - name: type-token-name
    decimals: 18
    address: type-token-address
    minToCollect: 1
  - name: type-token-name
    decimals: 18
    address: type-token-address
    minToCollect: 1
chainID: 56
chainRPCEndpoint: https://bsc-dataseed.binance.org/
APIPostDepositTransactionURL: https://example.com/user-deposited
APIPostTokensTransferredURL: https://example.com/transferred-tokens
ipWhiteList:
  - localhost
  - 192.168.1.0
APIKey: example-key
password: example-password
```

**Giải thích các trường:**

- _log_ `string` đường dẫn tới thư mực lưu log. (default: `./logs`)
- _debug_ `bool` true nếu ở chế độ debug, ở chế độ này sẽ ghi log chi tiết hơn để dễ dàng cho việc debug.
- _maxWallets_ `int` số lượng ví tối đa có thể tạo, ví dụ hệ thống chỉ cần tạo 1000 ví cho các user id từ 1 tới 1000 thì
  đặt maxWallets = 1000.
- _amqpURI_ `string` uri connect với rabbitmq, default là `amqp://guest:guest@andromeda-rabbitmq:5672/` là uri của
  rabbitmq chạy trong andromeda service, nếu muốn dùng rabbitmq khác chạy bên ngoài thì đặt lại trường này.
- _mongoDNS_ `string` uri tới mongodb, default là `mongodb://andromeda:andromeda@andromeda-mongodb:27017` là uri tới
  mongodb chạy trong andromeda service, nếu muốn móc sang mongo db khác chạy bên ngoài thì sửa lại trường này.
- _hotWalletMnemonic_ `string` chuỗi mnemonic (hay trên metamask gọi là **Cụm mật khẩu khôi phục bí mật**) của ví tổng,
  ví dụ 1 mnemonic: `mistake wet reward valid gesture drop delay soft say same energy upon`
- _numberOfConfirmationsToApproveDepositTx_ `int` số lượng tối thiểu xác nhận để giao dich có thể coi là hợp lệ, số này
  càng nhỏ thì khả năng cheat của user càng cao, thường sẽ từ 15 trở lên, tuy nhiên nếu để quá cao có thể thời gian của
  người chơi để chờ xác nhận giao dịch sẽ lâu hơn nhiều.
- _tokens_ `array` là mảng các loại token được hỗ trợ
    - _name_ `string` tên duy nhất của token (do mình tự đặt) (ví dụ: usdt, usdc, bnb, busd,...)
    - _decimals_ `int` decimal của token.
    - _address_ `string` địa chỉ của token.
    - _minToCollect_ `int` số lượng token tối thiểu có trong ví con để **andromeda** thực hiện rút tiền về ví tổng, bởi
      vì mỗi lần rút sẽ mất khoảng `0.2$` phí giao dịch, nên phải giới hạn rút đối với số lượng tiền tối thiểu, tránh số
      tiền quá nhỏ vẫn thực hiện rút dẫn đến tình trạng tiền không đủ bù phí giao dịch. Một user có thể nạp nhiều lần số
      nhỏ, khi đạt đủ số lượng hệ thống mới thực hiện rút tiền về ví tổng nhằm tiết kiệm chi phí.
- _chainID_ `id` của mạng (**56** với mainnet và **97** với mạng testnet)
- _chainRPCEndpoint_ `string` rpc endpoint của mạng (mainnet: `https://bsc-dataseed.binance.org/` ,
  testnet `https://data-seed-prebsc-1-s1.binance.org:8545/`)
- _APIPostDepositTransactionURL_ `string` đường dẫn
  tới [API Complete Deposit Transaction Redirect URL](#complete-player-deposit-transaction-redirect-url)
- _APIPostTokensTransferredURL_ `string` đường dẫn tới
  API [API Complete Transferred Tokens Redirect URL](#complete-transferred-tokens-redirect-url)
- _ipWhiteList_ `array` các địa chỉ IP hợp lệ có thể gửi request lên, nếu mảng này rỗng thì sẽ accept request của mọi
  địa chỉ IP.
- _APIKey_ `string` **API Key** của hệ thống, phải khớp ở cả 2 đầu **Andromeda Service** và **Game Service**, đây sẽ là
  mã khóa bí mật không được để tiết lộ ra ngoài.
- _password_ `string` là mật khẩu dùng để xác thực đơn giản trong API, nếu password được gán thì mỗi request đều phải
  gửi lên kèm password trong Header.

## Deployment

Andromeda gồm 3 thành phần chính: Cash In, Cash Out và API.
Để khởi chạy toàn hệ thống:

```shell
sudo docker compose up -d 
```

Hoặc chạy từng phần như sau:

**Bước 1:** Tạo file `config.yaml` trong thư mục `configs/` theo hướng dẫn ở phần trên.
**Bước 2:** Chạy API Service

```shell
sh  deploy-API-service.sh
```

**Bước 3:** Chạy Cash In

```shell
sh deploy-CI-services.sh
```

**Bước 4:** Chạy Cash Out

```shell
sh deploy-CO-services.sh
```

**Bước 5:** Tạo ví cho người dùng

```shell
sh create-user-wallet.sh
```

Nếu không dùng hệ điều hành Linux thì dùng lệnh sau:

```shell
sudo docker exec -it andromeda-api ./andromeda migrate --create-user-wallet --mongo-dns="mongodb://andromeda:andromeda@andromeda-mongodb:27017"
```

Bước 6: Kiểm tra các service đã chạy lên đầy đủ chưa:

```shell
sudo docker compose ps 
```

Kết quả sẽ hiện lên như sau

```shell
NAME                   COMMAND                  SERVICE             STATUS              PORTS
andromeda-api          "/usr/bin/supervisor…"   api                 running (healthy)   0.0.0.0:9711->9711/tcp, :::9711->9711/tcp
andromeda-approver     "/usr/bin/supervisor…"   approver            running (healthy)   
andromeda-collector    "/usr/bin/supervisor…"   collector           running (healthy)   
andromeda-dispatcher   "/usr/bin/supervisor…"   dispatcher          running (healthy)   
andromeda-mongodb      "docker-entrypoint.s…"   mongo               running (healthy)   0.0.0.0:9707->27017/tcp, :::9707->27017/tcp
andromeda-rabbitmq     "docker-entrypoint.s…"   rabbitmq            running (healthy)   25672/tcp
andromeda-transactor   "/usr/bin/supervisor…"   transactor          running (healthy)   
andromeda-validator    "/usr/bin/supervisor…"   validator           running (healthy)   
andromeda-watcher      "/usr/bin/supervisor…"   watcher             running (healthy)  
```

Tình trạng các service đều ở trạng thái `healthy` là ok.

Port của API sẽ là port `9711` và port của mongo db là `9707` username và password là `andromeda`.

**Hệ thống gồm các thành phần sau đây:**

- **Cở sở dữ liệu:** mongo db, mở port `9707` ra ngoài, _username_: **andromeda**, _password_: **andromeda**.
    - _docker compose service_: mongo
    - _image_: mongo
    - _container_: andromeda-mongo
    - _run_: ```sh restart-mongodb.sh```
- **Message Broker:** rabbitmq
    - _docker compose service_: rabbitmq
    - _image_: rabbitmq:3.8.25-management-alpine
    - _container_: andromeda-rabbitmq
    - _run_: ```sh restart-rabbitmq.sh```
- **Cash In:** thực hiện ghi nhận các giao dịch nạp tiền vào hệ thống và bắn thông báo sang Game Service.
    - **module watcher** thực hiện lắng nghe các giao dịch trên mạng.
        - _docker compose service_: watcher
        - _image_: andromeda/watcher:v1.0.0
        - _container_: andromeda-watcher
        - _run_: ```sh restart-watcher.sh```
    - **module validator** thực hiện xác thực giao dịch có đúng thuộc user của hệ thống hay không
        - _docker compose service_: validator
        - _image_: andromeda/validator:v1.0.0
        - _container_: andromeda-validator
        - _run_: ```sh restart-validator.sh```
    - **module approver** thực hiện kiểm tra giao dịch đã đạt đủ số lượng xác nhận từ các node trên mạng chưa, để tránh
      hack.
        - _docker compose service_: approver
        - _image_: andromeda/approver:v1.0.0
        - _container_: andromeda-approver
        - _run_: ```sh restart-approver.sh```
    - **module dispatcher** nhận các yêu cầu vào gửi thông báo qua API sang bên Game Service.
        - _docker compose service_: dispatcher
        - _image_: andromeda/dispatcher:v1.0.0
        - _container_: andromeda-dispatcher
        - _run_: ```sh restart-dispatcher.sh```
    - **module collector** thực hiện thu hồi token tự động từ các ví con về ví tổng.
        - _docker compose service_: collector
        - _image_: andromeda/collector:v1.0.0
        - _container_: andromeda-collector
        - _run_: ```sh restart-collector.sh```
- **Cash Out:** thực hiện lấy các yêu cầu rút tiền về tài khoản người dùng và chuyển tiền vào ví của người dùng.
    - **module transactor** nhận yêu cầu rút tiền từ hệ thống và thực hiện chuyển tiền sang ví của user.
        - _docker compose service_: transactor
        - _image_: andromeda/transactor:v1.0.0
        - _container_: andromeda-transactor
        - _run_: ```sh restart-transactor.sh```
- **API Server**: cung cấp các API để Game Service có thể làm việc với **Andromeda**, port `9711`.
    - _docker compose service_: api
    - _image_: andromeda/api:v1.0.0
    - _container_: andromeda-api
    - _run_: ```sh restart-api.sh```
- **Data:** thư mục `data/` chứa dữ liệu của database và rabbitmq, không được thay đổi quyền của thư mục và không được
  xóa thư mục nếu không dữ liệu sẽ bị mất.
- **Log:** thư mục `logs/`chứa log của hệ thống, log sẽ ghi theo ngày và sẽ có tên của module ứng với file log đó. Ví dụ
  file `2022-07-19.api.log` chứa log của API vào ngày 19 tháng 7 năm 2022. File `stdout.log` ghi log std.
- **Tạo ví:** Sau khi khởi động hệ thống lần đầu tiên, cơ sở dữ liệu sẽ chưa có thông tin ví của người dùng, mà ta phải
  tạo ví bằng tay. Lệnh sẽ sử dụng **Mnemonic** trong [file config](#config) làm ví tổng để tạo ví con và **MaxWallets**
  trong [config](#config) để xác định số lượng tối đa ví sẽ tạo.
    - _Run:_ ```sh create-user-wallet.sh```
      hoặc ```sudo docker exec -it andromeda-api ./andromeda migrate --create-user-wallet --mongo-dns="mongodb://andromeda:andromeda@andromeda-mongodb:27017"```
- **Dọn dẹp Watched Block:** Nếu hệ thống chạy sai hoặc thay đổi Chain thì phải Clean lại bảng _watched_block_ trong DB
  vì bảng này lưu vị trí block gần nhất crawl được từ Chain, mà mỗi chain thì số block khác nhau nên phải clean để tự
  cập nhật lại, thao tác này có thể dẫn đến mất mát dữ liệu trong khoảng thời gian ngắn.
    - _Run:_ ```sh clean-watched-block.sh```

## Run Test

```shell
sudo docker exec -it andromeda-api ./andromeda test --amount-per-transaction=0.001 --number-of-transactions=10 --private-key=bde426c3c03bf9254c629c3a1407c0278fa7ddd9e13b6ffdec7f5fa4e8f69069 --token=usdt
```

**Flags:**

- _amount-per-transaction_ `float` số tiền gửi trên 1 giao dịch.
- _number-of-transactions_ `int` số lượng giao dịch.
- _private-key_ `string` private key của tài khoản dùng để gửi tiền.
- _token_ `string` loại tiền gửi.

## Database

### deposit_transactions

Bảng lưu thông tin các giao dịch.

### withdraw_transactions

### users

### last_watched_block

### failed_request

## API

**Lưu ý:** Nếu Password được set thì khi call API bắt buộc phải đặt trường `Authorization` trong header theo cú
pháp `"Password " + password`.
Ví dụ nếu password là **"example"** thì trường Authorization sẽ là `"Password example"`.

Đối với tất cả các API bên dưới thì sẽ trả về các Status Code chung sau:

- **406** `Not Acceptable` đối với địa chỉ IP không nằm trong white list.
- **401** `Unauthorized` đối với password không được gắn trong Header hoặc không trùng với password hệ thống.

### Complete Player Deposit Transaction Redirect URL

Sau khi phát hiện giao dịch nạp tiền và xác thực giao dịch, Andromeda sẽ gửi thông báo sang
Game Service để thông báo giao dịch nạp tiền đã hoàn tất.

Game Service phải cung cấp API để Andromeda gửi thông báo sang, Andromeda sẽ thực hiện tôi đa 3 lần gọi với mỗi
giao dịch ghi nhận được, nếu vượt quá số lượng trên, gói tin sẽ được lưu trữ vào trong cơ sở dữ liệu.

**Thông tin API:**

- _API URL:_ tự khai báo ở trong [Config](#config), ứng với API viết bên **Game Service**. (ví
  dụ: https://game.service.com/api/v1/user-deposited)
- _Method:_ POST
- _Body_:
    - _user_id_ `int` ID của người chơi
    - _tx_hash_ `string` mã giao dịch
    - _tokens_ `float` số lượng tokens nạp.
    - _token_name_ `string` tên token.
    - _token_address_ `string` địa chỉ token.
    - _decimals_ `int` decimals của token (thường là 18)
    - _chain_id_ `int` id của chain nạp (**56** với mạng **mainnet**, **97** với mạng **testnet**)
    - _check_sum_ `string` bằng `MD5(API_KEY + TxHash)` dùng để xác thực request này có thật sự tới từ **Andromeda
      Service** không, vì nếu tới từ bên ngoài thì sẽ không thể dò ra được `check_sum` tương ứng với `tx_hash` do không
      thể biết được `API_KEY`.

**Lưu ý:**

- **Game Service** phải lưu lại **tx_hash** vào **database** để kiểm tra xem giao dịch này đã được cập nhật trước đó
  chưa, tránh trường hợp bên thứ 3 tấn công dùng lại gói tin này gửi lên nhằm trục lợi nạp nhiều lần từ 1 giao dịch.
- _API URL_ để là `host.docker.internal` thay cho `localhost` nếu **Andromeda Service** và **Game Service** chạy trong
  cùng 1 máy chủ.
- _API URL_ nên để **HTTPS** để tránh kẻ tấn công có thể lấy được thông tin của gói tin.
- **Bắt buộc** phải kiểm tra **check_sum** để xác thực gói tin này có thật sự tới từ **Andromeda Service** hay không.
- Ví dụ _API Key_ là `abc` và _Tx Hash_ là `0x2d157578aee3908b41192bad9af19432eb683ea872152fa3c72a2ba7a4095abb`  thì
  check_sum sẽ là hàm băm MD5 của chuỗi `abc0x2d157578aee3908b41192bad9af19432eb683ea872152fa3c72a2ba7a4095abb` và
  là `ea16990b87483afaab3a4740a47a2c98`
- Có thể thực hiện kiểm tra địa chỉ IP gửi gói tin này đến **Game Service** để tăng thêm tính bảo mật cho hệ thống.

**!Lưu ý: Mỗi khi nhận được request này từ andromeda, lưu giao dịch vào database và kiểm tra lại mỗi lần nhận được request
xem tx_hash này đã được insert trước đó chưa để tránh trường hơp hacker có thể tận dụng lại request để gửi nhiều lần để
chuộc lợi từ 1 giao dịch.**

### Complete Transferred Tokens Redirect URL

Sau khi chuyển tiền thành công vào ví người nhận, andromeda sẽ gửi thông báo sang 1 API Complete Transferred Tokens
Redirect.

Game Service cung cấp API này để andromeda thông báo sang, nếu trong [config](#config) để trường _
APIPostTokensTransferredURL_
là rỗng thì andromeda sẽ không thông báo nữa.

**Cấu trúc:**

- **URL**: định nghĩa trong file config
- **Method**: `POST`
- **Body**:
    - _request_id_ `string` request id mà game service gửi lên trong [API Withdraw](#withdraw)
    - _user_id_ `int` ID của người chơi
    - _tx_hash_ `string` mã giao dịch trên blockchain.
    - _tokens_ `float` số tiền đã gửi.
    - _token_name_ `string` tên tiền.
    - _checksum_ `string`  = MD5(API-Key + Request ID) để kiểm tra độ tin cậy của request.

**!Lưu ý: Mỗi khi nhận được request này từ andromeda, lưu giao dịch vào database và kiểm tra lại mỗi lần nhận được request
xem request_id này đã được complete trước đó chưa để tránh trường hơp hacker có thể tận dụng lại request để gửi nhiều lần để
chuộc lợi từ 1 giao dịch.**

### Get User Address

Lấy địa chỉ ví của user.

```http request
GET /api/v1/user/:userID/address
```

Request:

- _userID_ `int` là id của user cần lấy địa chỉ ví, sẽ trả về lỗi nếu userID <= 0.

Ví dụ lấy địa chỉ ví user có id 10.

```http request
GET /api/v1/user/10/address
```

**Response:**

- _user_id_ `int` ID của user
- _address_ `string` địa chỉ ví

**Status Code:**

- **400** `Bad Request` user id truyền lên không phải dạng số hoặc không nằm trong khoảng cho phép từ 1 tới 4294967295.
- **404** `Not Found` lỗi xảy ra khi mnemonic trong config bị sai dẫn đến thuật toán không thể tìm ra địa chỉ ví.
- **200** `OK` success

### User Count

Trả về số lượng ví đã tạo

```http request
GET /api/v1/user/count
```

**Response:**

- _count_ `int` số lượng ví đã tạo.

**Status Code:**

- **404** `Not Found` truy vấn cơ sở dữ liệu thất bại.
- **200** `OK` success

### Get User ID

Trả về User ID ứng với địa chỉ ví

```http request
GET /api/v1/user/:address/id
```

Ví dụ: đối với địa chỉ ví `0xB6330d4350cb2b3b8A4776eEFFA4D67299915F84`

```http request
GET /api/v1/user/0xB6330d4350cb2b3b8A4776eEFFA4D67299915F84/id
```

**Request:**

- **address** `string` địa chỉ ví, trả về lỗi nếu địa chỉ không hợp lệ.

**Response:**

- _user_id_ `int` id của user.
- _address_ `string` địa chỉ ví.

**Status Code:**

- **400** `Bad Request` address không phải là địa chỉ ví.
- **404** `Not Found` không tìm thấy địa chỉ ví này trong cơ sở dữ liệu.
- **200** `OK` success.

### Get User Deposit Transactions Count

Trả về số lượng tổng giao dịch nạp tiền đã thành công của user

```http request
GET /api/v1/:userID/deposit-transactions/count
```

**Request:**

- _userID_ `int` ID của người chơi

**Response:**

- _user_id_ `int` id của người chơi
- _count_ `int` số lượng giao dịch
-

### Get User Completed Deposit Transactions

Lấy các giao dịch nạp tiền đã hoàn thành của người chơi.

```http request
GET /api/v1/user/:userID/deposit-transactions/:pageSize/:page
```

**Request:**

- _userID_ `int` id của người chơi.
- _pageSize_ `int` số lượng giao dịch tối đa lấy.
- _page_ `int` vị trí của page lấy, ví dụ _page_ = 2, _pageSize_ = 10 thì lấy giao dịch thứ 10 tới thứ 20.

**Response:**

- [DepositTransaction[]](#deposittransaction) danh sách giao dịch.

**Status Code:**

- **400** `Bad Request` user ID, page size, page không phải là số hoặc không hợp lệ.
- **500** `Internal Server Error` không thể thực hiện truy vấn cơ sở dữ liệu.
- **200** `OK` success

### Get User Pending Deposit Transactions

Lấy tất cả các giao dịch nạp tiền đang chờ xác thực.

```http request
GET /api/v1/user/:userID/pending-deposit-transactions
```

**Request:**

- _userID_ `int` id của người chơi.

**Response:**

- [DepositTransaction[]](#deposittransaction) danh sách giao dịch.

**Status Code:**

- **400** `Bad Request` user ID không phải là số hoặc không hợp lệ.
- **500** `Internal Server Error` không thể thực hiện truy vấn cơ sở dữ liệu.
- **200** `OK` success

### Get User Withdraw Transactions Count

Trả về số lượng tổng giao dịch rút tiền đã thành công của user

```http request
GET /api/v1/:userID/withdraw-transactions/count
```

**Request:**

- _userID_ `int` ID của người chơi

**Response:**

- _user_id_ `int` id của người chơi
- _count_ `int` số lượng giao dịch

### Get User Completed Withdraw Transactions

Lấy các giao dịch rút tiền đã hoàn thành của người chơi.

```http request
GET /api/v1/user/:userID/withdraw-transactions/:pageSize/:page
```

**Request:**

- _userID_ `int` id của người chơi.
- _pageSize_ `int` số lượng giao dịch tối đa lấy.
- _page_ `int` vị trí của page lấy, ví dụ _page_ = 2, _pageSize_ = 10 thì lấy giao dịch thứ 10 tới thứ 20.

**Response:**

- [WithdrawTransaction[]](#withdrawtransaction) danh sách giao dịch.

**Status Code:**

- **400** `Bad Request` user ID, page size, page không phải là số hoặc không hợp lệ.
- **500** `Internal Server Error` không thể thực hiện truy vấn cơ sở dữ liệu.
- **200** `OK` success

### Get User Pending Withdraw Transactions

Lấy tất cả các giao dịch rút tiền đang chờ thực hiện.

```http request
GET /api/v1/user/:userID/withdraw-deposit-transactions
```

**Request:**

- _userID_ `int` id của người chơi.

**Response:**

- [WithdrawTransaction[]](#withdrawtransaction) danh sách giao dịch.

**Status Code:**

- **400** `Bad Request` user ID không phải là số hoặc không hợp lệ.
- **500** `Internal Server Error` không thể thực hiện truy vấn cơ sở dữ liệu.
- **200** `OK` success

### Withdraw

Yêu cầu hệ thống thực hiện yêu cầu rút tiền của người chơi và chuyển token sang ví của người chơi.

```http request
POST /api/v1/withdraw 
```

**Request Body:**

- _user_id_ `int` id của người chơi
- _address_ `string` địa chỉ ví
- _token_name_ `string` tên token
- _amount_ `float` số lượng token cần rút.
- _request_id_ `string` ID của giao dịch bên game service, ví dụ như ID của giao dịch lưu trong DB bên game service hoặc
  gì
  đó tương tự để hệ thống không thực hiện 1 giao dịch quá 1 lần, trong trường hợp game service gửi nhầm 2 lần 1 gói tin.
- _checksum_ `string` `MD5(API-Key + request_id)` mã checksum dùng để phát hiện gian lận nếu người gửi request này không
  phải Game Service.

**Status Code:**

- **400** `Bad Request` dữ liệu gửi lên không hợp lệ.
- **500** `Internal Server Error` không thể khởi tạo giao dịch.
- **201** `Created` đã khởi tạo thành công giao dịch rút tiền.

### Model

#### DepositTransaction

- _id_ `string` ID của record.
- _tx_hash_ `string` mã giao dịch.
- _user_id_ `int` ID của user.
- _contract_address_ `string` địa chỉ contract.
- _from_address_ `string` địa chỉ ví gửi.
- _to_address_ `string` địa chỉ ví con nhận tiền.
- _block_number_ `int` số thứ tự block.
- _block_hash_ `string` mã block.
- _value_ `string` tiền khi chưa chia cho decimal.
- _token_name_ `string` tên của token khai báo trong file config.
- _token_address_ `string` địa chỉ của token.
- _tokens_ `float` số tiền gửi.
- _chain_id_ `int` id của mạng.
- _decimals_ `int`
- _is_approved_ `bool` trường này bằng true nếu giao dịch đã được xác thực thành công, còn false nếu giao dịch đang đợi
  hoàn thành.
- _is_collected_ `bool` nếu tiền đã được chuyển về ví tổng thì = true.
- _updated_at_ `int` unix timestamp .
- _created_at_ `int` unix timestamp.
- _max_confirmations_ `int` số lượng tối đa xác nhận cần cho giao dịch này (chỉ dùng trong trường hợp pending
  transaction)
- _confirmations_ `int` số lượng xác nhận hiện tại (nếu = _max_confirmations_ thì giao dịch đã thành công), trong giao
  diện lịch sử giao dịch sẽ hiển thị số (confirmations / max confirmations) đối với các giao dịch đang thực hiện, để
  người dùng có thể biết là giao dịch này tiến độ hoàn thành đang là bao nhiêu %.

#### WithdrawTransaction

- _id_ `string` ID của record.
- _request_id_ `string` id của request này ứng với bên game server, để map 2 transaction với nhau.
- _user_id_ `int` id của user.
- _tokens_ `float` số tiền yêu cầu rút.
- _token_name_ `string` tên token.
- _token_address_ `string` địa chỉ token.
- _to_address_ `string` địa chỉ ví nhận tiền.
- _is_transferred_ `bool` true nếu tiền đã được gửi sang ví người dùng.
- _tx_hash_ `string` mã giao dịch (nếu _is_transferred_ = true).
- _created_at_ `int` unix timestamp
- _updated_at_ `int` unix timestamp 