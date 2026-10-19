# Triển khai MegaBotAI trên Render

Dashboard trong `main/manager-web` đã có giao diện tiếng Việt. Khi mở lần đầu, hệ thống dùng tiếng Việt; người dùng vẫn có thể đổi ngôn ngữ trong giao diện. Mã server gốc nằm nguyên trong `main/xiaozhi-server`.

## Các dịch vụ cần có

Server hiện gồm hai tiến trình giao tiếp với firmware: WebSocket trên cổng `8000` và HTTP OTA trên cổng `8003`. Firmware gọi `POST /xiaozhi/ota/` để nhận thông tin kết nối, sau đó mở WebSocket tại `/xiaozhi/v1/`. Dashboard quản trị dùng ứng dụng Vue và API Java; chế độ đầy đủ cần MySQL và Redis.

Render chỉ chuyển tiếp lưu lượng Internet công khai đến một cổng HTTP cho mỗi Web Service, bao gồm cả kết nối WebSocket. Vì vậy cần một reverse proxy công khai định tuyến `/xiaozhi/v1/` tới cổng WebSocket `8000`, `/xiaozhi/ota/` và `/mcp/vision/explain` tới HTTP `8003` ([giới hạn cổng](https://render.com/docs/web-services), [WebSocket trên Render](https://render.com/docs/websocket)). Cấu hình Docker hiện có trong repo dành cho Docker Compose và không tự thực hiện ánh xạ cổng kiểu này trên Render. Dashboard/API và server thoại có thể chạy thành các dịch vụ riêng; MySQL/Redis phải có thể truy cập từ các dịch vụ đó.

## Cấu hình firmware

Sau khi tạo dịch vụ công khai và có hostname HTTPS trên Render:

1. Tạo file `data/.config.yaml` từ `main/xiaozhi-server/config.yaml` để giữ cấu hình riêng bên ngoài mã gốc.
2. Điền `server.websocket` bằng `wss://<hostname>/xiaozhi/v1/` và `server.vision_explain` bằng `https://<hostname>/mcp/vision/explain`.
3. Trong `fw-megabotai`, chạy `idf.py menuconfig`, mở **Xiaozhi Assistant → Default OTA URL** và đặt `https://<hostname>/xiaozhi/ota/`; sau đó build đúng board/variant bằng `python scripts/build.py <board-directory> --name <variant-name>`.
4. Nạp firmware, kết nối Wi-Fi rồi kiểm tra log thiết bị. OTA trả về URL WebSocket của server và thiết bị dùng URL đó để mở kênh âm thanh.

> Thay `<hostname>` bằng hostname thật trước khi build. Không dùng URL `localhost`, IP nội bộ hoặc hostname ví dụ cho thiết bị chạy ngoài mạng server.

## Biến môi trường và dữ liệu

- Không đưa API key, mật khẩu cơ sở dữ liệu hoặc `data/.config.yaml` có bí mật lên GitHub.
- Dùng secret variables của Render cho thông tin kết nối MySQL/Redis, API secret và khóa dịch vụ mô hình.
- Gắn persistent disk cho dữ liệu server, log, file tải lên và dữ liệu MySQL nếu tự vận hành MySQL. Không dùng ổ đĩa tạm cho dữ liệu cần giữ qua lần deploy.
- Cấu hình `server.websocket` bằng hostname HTTPS công khai; reverse proxy phải hỗ trợ WebSocket Upgrade và giữ kết nối lâu dài.
- Bật xác thực server và giới hạn thiết bị trước khi đưa endpoint OTA/API ra Internet.

GitHub lưu mã nguồn và kích hoạt build/deploy khi có push; GitHub không tự chạy container. Render cần được liên kết với repository và cấu hình các dịch vụ/biến môi trường ở phía Render trước khi firmware có thể kết nối đến server đang chạy.

## Kiểm tra sau deploy

- Mở `https://<dashboard-hostname>/` và xác nhận giao diện hiển thị tiếng Việt.
- Kiểm tra API quản trị và trạng thái kết nối database/Redis trong log dịch vụ.
- Gửi yêu cầu OTA từ firmware hoặc client kiểm thử tới `/xiaozhi/ota/`; xác nhận phản hồi có trường `websocket.url`.
- Kiểm tra WebSocket `wss://<hostname>/xiaozhi/v1/` qua reverse proxy, sau đó thử hội thoại bằng thiết bị thật.
- Xác nhận upload firmware/model, log và dữ liệu cấu hình vẫn còn sau khi redeploy.
