# FHEAuction: Đấu giá Mù (Blind Auction) Bảo mật Toàn diện

[![Giấy phép: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Build Status](https://img.shields.io/travis/ci/[username-github]/[tên-repo].svg)](https://travis-ci.org/[username-github]/[tên-repo])
[![Powered by FHEVM](https://img.shields.io/badge/Powered%20by-FHEVM-blue.svg)](https://www.zama.ai/fhevm)
[![Solidity Version](https://img.shields.io/badge/Solidity-%5E0.8.24-lightgrey.svg)](https://soliditylang.org/)

> Một hợp đồng đấu giá mù phi tập trung, bảo mật trên Ethereum, sử dụng **Fully Homomorphic Encryption (FHE)** để đảm bảo tính riêng tư tuyệt đối cho các giá thầu.



Dự án này triển khai một phiên đấu giá nơi người tham gia có thể đặt thầu mà không tiết lộ giá trị của họ cho bất kỳ ai—kể cả quản trị viên hợp đồng—cho đến khi phiên đấu giá kết thúc. Tất cả các so sánh giá thầu (để tìm ra giá thầu cao nhất) được thực hiện **trên dữ liệu đã mã hóa** bằng cách sử dụng `FHE.gt()` và `FHE.select()` từ thư viện FHEVM của Zama.

---

## 📚 Mục lục

* [Kiến trúc & Tính năng](#-kiến-trúc--tính-năng)
* [Luồng hoạt động (Workflow)](#-luồng-hoạt-động-workflow)
* [Hướng dẫn cho Người dùng dApp (Client-Side)](#-hướng-dẫn-cho-người-dùng-dapp-client-side)
    * [Cài đặt (NPM)](#cài-đặt-npm)
    * [Ví dụ: Đặt thầu (Javascript)](#ví-dụ-đặt-thầu-javascript)
* [Hướng dẫn cho Nhà phát triển (Solidity)](#-hướng-dẫn-cho-nhà-phát-triển-solidity)
    * [Yêu cầu hệ thống](#yêu-cầu-hệ-thống)
    * [Cài đặt & Chạy Local](#cài-đặt--chạy-local)
    * [Kiểm thử (Testing)](#kiểm-thử-testing)
    * [Triển khai (Deployment)](#triển-khai-deployment)
* [API Hợp đồng (Chức năng chính)](#-api-hợp-đồng-chức-năng-chính)
* [An toàn & Bảo mật](#-an-toàn--bảo-mật)
* [Đóng góp](#-đóng-góp)
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

## 💻 Hướng dẫn cho Người dùng dApp (Client-Side)

Đây là cách tích hợp và tương tác với hợp đồng FHEAuction từ một ứng dụng Javascript/TypeScript (ví dụ: React, Next.js, Vue).

### Cài đặt (NPM)

Bạn sẽ cần `ethers` (hoặc `viem`) để tương tác với blockchain và `@fhevm/sdk` để mã hóa giá thầu.

```bash
npm install ethers @fhevm/sdk
# hoặc
yarn add ethers @fhevm/sdk
