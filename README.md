# FHEAuctionV3: Đấu giá Mù (Blind Auction) Bảo mật Toàn diện

[![Giấy phép: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Powered by FHEVM](https://img.shields.io/badge/Powered%20by-FHEVM-blue.svg)](https://www.zama.ai/fhevm)
[![Solidity Version](https://img.shields.io/badge/Solidity-%5E0.8.24-lightgrey.svg)](https://soliditylang.org/)

> Một hợp đồng đấu giá mù phi tập trung, bảo mật trên Ethereum, sử dụng **Fully Homomorphic Encryption (FHE)** để đảm bảo tính riêng tư tuyệt đối cho các giá thầu.



Dự án này triển khai một phiên đấu giá nơi người tham gia có thể đặt thầu mà không tiết lộ giá trị của họ cho bất kỳ ai—kể cả quản trị viên hợp đồng—cho đến khi phiên đấu giá kết thúc. Tất cả các so sánh giá thầu (để tìm ra giá thầu cao nhất) được thực hiện **trên dữ liệu đã mã hóa** bằng cách sử dụng `FHE.gt()` và `FHE.select()` từ thư viện FHEVM của Zama.

---

## 📚 Mục lục

* [Kiến trúc & Tính năng](#-kiến-trúc--tính-năng)
* [Luồng hoạt động (Workflow)](#-luồng-hoạt-động-workflow)
* [Hướng dẫn cho Người dùng dApp (Client-Side)](#-hướng-dẫn-cho-người-dùng-dapp-client-side)
* [Hướng dẫn cho Nhà phát triển (Solidity)](#-hướng-dẫn-cho-nhà-phát-triển-solidity)
    * [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
    * [Cài đặt & Chạy Local](#cài-đặt--chạy-local)
    * [Triển khai (Deployment)](#triển-khai-deployment)
* [API Hợp đồng (Chức năng chính)](#-api-hợp-đồng-chức-năng-chính)
* [An toàn & Bảo mật](#-an-toàn--bảo-mật)
* [Giấy phép](#-giấy-phép)

---

## ✨ Kiến trúc & Tính năng

* **Bảo mật bởi FHE:** Giá thầu được mã hóa client-side (thành `euint64`) và không bao giờ được giải mã on-chain trong suốt quá trình đấu giá.
* **Đấu giá Mù Thực sự:** Không ai có thể thấy giá thầu của người khác cho đến khi phiên đấu giá kết thúc, ngăn chặn "front-running" và "bid sniping".
* **So sánh Đồng hình:** Hợp đồng sử dụng `FHE.gt()` (lớn hơn) và `FHE.select()` (chọn) để tìm ra giá thầu cao nhất mới (`encryptedMaxBid`) hoàn toàn trên dữ liệu đã mã hóa.
* **Xác thực sau Giải mã:** Để duy trì tính riêng tư, các quy tắc (như `MIN_BID_INCREMENT`) chỉ được kiểm tra *sau khi* phiên đấu giá kết thúc và KMS đã giải mã các giá thầu. Các giá thầu không hợp lệ sẽ được hoàn tiền 100%.
* **Cơ chế Chống-Snipe:** Tự động gia hạn thời gian đấu giá (`EXTENSION_DURATION_BLOCKS`) nếu có giá thầu được đặt gần cuối.
* **Bảo mật Nâng cao:** Tích hợp `ReentrancyGuard`, EIP-712, xác thực `onlyGateway` cho callback KMS, và cơ chế hoàn tiền "Pull-over-Push".
* **Cơ chế Trạng thái (State Machine):** Quản lý vòng đời đấu giá qua 5 trạng thái rõ ràng (`Active`, `Ended`, `Finalizing`, `Finalized`, `Emergency`).

---

## 🔄 Luồng hoạt động (Workflow)

1.  **Giai đoạn 1: Đặt thầu (Active)**
    * Người dùng (Bob) quyết định đặt thầu `100 Gwei`.
    * Client-side dApp của Bob sử dụng `@fhevm/sdk` để mã hóa `100` thành `encryptedBid`.
    * Bob gọi hàm `bid(encryptedBid, ...)` và gửi kèm `msg.value` (tiền ký gửi, phải >= `minBidDeposit`).
    * Hợp đồng cập nhật `encryptedMaxBid` bằng phép toán FHE.

2.  **Giai đoạn 2: Kết thúc (Ended)**
    * `block.number` vượt qua `auctionEndBlock`. Giai đoạn đặt thầu kết thúc.

3.  **Giai đoạn 3: Yêu cầu giải mã (Finalizing)**
    * Chủ sở hữu (owner) gọi `requestFinalize()`.
    * Hợp đồng thu thập tất cả các `encryptedBid` và gửi yêu cầu giải mã đến Zama KMS Gateway.

4.  **Giai đoạn 4: Callback & Xác thực**
    * KMS Gateway giải mã các giá thầu và gọi lại hàm `onDecryptionCallback()` với các giá trị plaintext.
    * Hợp đồng xác thực chữ ký của KMS.

5.  **Giai đoạn 5: Hoàn tất (Finalized)**
    * Hợp đồng *lúc này mới* lặp qua các giá thầu plaintext, kiểm tra `MIN_BID_INCREMENT`, và xác định người thắng cuộc hợp lệ.
    * Chuyển tiền cho người hưởng lợi (`beneficiary`) và thu phí (`feeCollector`).
    * Người thua và người đặt thầu không hợp lệ nhận được `pendingRefunds` (tiền chờ hoàn).
    * Một vòng đấu giá mới (`currentRound + 1`) tự động bắt đầu.

---

## 🛠️ Hướng dẫn cho Nhà phát triển (Solidity)

Phần này dành cho các nhà phát triển muốn fork, kiểm thử, hoặc triển khai hợp đồng `FHEAuctionV3`. Dự án này được xây dựng bằng Hardhat.

### Yêu cầu hệ thống

* [Node.js](https://nodejs.org/) (v18+)
* [Yarn](https://yarnpkg.com/) (khuyến nghị) hoặc `npm`
* [Git](https://git-scm.com/)

### Cài đặt & Chạy Local

1.  **Clone repo:**
    ```bash
    git clone [https://github.com/](https://github.com/)[username-github]/[tên-repo].git
    cd [tên-repo]
    ```

2.  **Cài đặt dependencies:**
    ```bash
    yarn install
    # hoặc
    npm install
    ```

3.  **Biên dịch hợp đồng:**
    ```bash
    npx hardhat compile
    ```

### Triển khai (Deployment)

Bạn phải triển khai hợp đồng này trên một mạng lưới hỗ trợ FHEVM (ví dụ: Zama Sepolia Testnet).

1.  **Cấu hình `.env`:**
    Tạo file `.env` và thêm Private Key của ví deploy:
    ```
    PRIVATE_KEY="0x..."
    SEPOLIA_RPC_URL="https://[rpc-url-cua-ban]"
    ```

2.  **Cấu hình `hardhat.config.ts`:**
    Đảm bảo bạn đã thêm mạng Zama Sepolia:
    ```typescript
    const config: HardhatUserConfig = {
      // ...
      networks: {
        zamaSepolia: {
          url: "[https://devnet.zama.ai](https://devnet.zama.ai)", // RPC của Zama
          accounts: [process.env.PRIVATE_KEY || ''],
        },
      },
    };
    ```

3.  **Chuẩn bị Script Triển khai (`deploy.ts`):**
    Bạn cần cung cấp các tham số cho `constructor`:
    * `_minDeposit`: Tiền ký gửi tối thiểu (ví dụ: `0.1 ETH`).
    * `_pauserSet`: Địa chỉ hợp đồng `IPauserSet` (hoặc `address(0)` nếu không dùng).
    * `_beneficiary`: Địa chỉ nhận tiền thắng.
    * `_gatewayContract`: **QUAN TRỌNG:** Địa chỉ Gateway của Zama.
        * *Sepolia Testnet: `0xa02Cda4Ca3a71D7C46997716F4283aa851C28812`*
    * `_feeCollector`: Địa chỉ nhận phí nền tảng.

4.  **Chạy lệnh Deploy:**
    ```bash
    npx hardhat deploy --network sepolia     
    ```

---

## 📜 API Hợp đồng (Chức năng chính)

Các hàm quan trọng nhất dành cho người dùng và quản trị viên.

### Chức năng cho Người dùng (Bidders)

* **`bid(externalEuint64 encryptedBid, bytes calldata inputProof, bytes32 publicKey, bytes calldata signature)`**
    * Hàm `payable` để đặt thầu. Gửi giá thầu đã mã hóa (từ FHE-SDK) và tiền ký gửi (`msg.value`).
* **`cancelBid()`**
    * Cho phép hủy thầu và nhận lại 100% tiền ký gửi *trước khi* phiên đấu giá kết thúc.
* **`withdrawRefund()`**
    * *(Giả định hàm này tồn tại dựa trên `pendingRefunds`)*: Rút lại tiền hoàn (nếu bạn thua hoặc giá thầu không hợp lệ) sau khi phiên đấu giá kết thúc.

### Chức năng cho Chủ sở hữu (Owner)

* **`requestFinalize()`**
    * Kích hoạt quá trình kết thúc và giải mã. Chỉ owner mới có thể gọi nếu có người đặt thầu.
* **`cancelDecryption()`**
    * Hàm khẩn cấp để hủy yêu cầu giải mã nếu KMS bị kẹt (sau thời gian `DECRYPTION_TIMEOUT_BLOCKS`).
* **`setBeneficiary(address _newBeneficiary)`**
    * Cập nhật địa chỉ nhận tiền thắng.
* **`setFeeCollector(address _newCollector)`**
    * Cập nhật địa chỉ nhận phí.
* **`pause()` / `unpause()`**
    * Tạm dừng/tiếp tục hợp đồng (cục bộ).

---

## 🛡️ An toàn & Bảo mật

Hợp đồng này tích hợp nhiều tính năng bảo mật tiêu chuẩn và nâng cao:

* **ReentrancyGuard:** `nonReentrant` modifier trên các hàm thay đổi trạng thái chính.
* **EIP-712:** Xác thực chữ ký để liên kết public key FHE với địa chỉ Ethereum.
* **Gateway Authentication:** `onlyGateway` modifier đảm bảo chỉ Zama KMS mới có thể gửi kết quả giải mã.
* **Pull-over-Push:** Sử dụng `pendingRefunds` mapping để người dùng tự rút tiền an toàn.
* **Custom Errors:** Tiết kiệm gas và cung cấp thông báo lỗi rõ ràng.
* **State Machine:** Ngăn chặn các hành động không hợp lệ (ví dụ: không thể `bid` khi đang `Finalizing`).

---

## ⚖️ Giấy phép

Dự án này được cấp phép theo **Giấy phép MIT**. Xem file `LICENSE` để biết chi tiết.
