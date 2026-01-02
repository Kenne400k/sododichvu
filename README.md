# Sơ đồ Kiến trúc Server KingCong

## 1. Sơ đồ Tổng quan Hệ thống

```mermaid
graph TB
    User[👤 Người dùng] -->|Gửi yêu cầu| FE[Giao diện - Server KingCong]
    FE --> BE[Backend - Bộ phân phối Task]
    
    BE --> CM[Quản lý Credits]
    BE --> TM[Quản lý Task]
    BE --> SM[Giám sát Trạng thái]
    
    CM -->|Kiểm tra Credits| DB[(Cơ sở dữ liệu<br/>Credits & Sử dụng)]
    
    TM -->|Phân phối Task| LB[Bộ cân bằng tải]
    
    LB -->|70% Task| AI33[Server AI33]
    LB -->|30% Task| AI84[Server AI84]
    
    AI33 --> EL33[ElevenLabs AI33<br/>Tối đa 30 luồng/key<br/>10 request/phút]
    AI33 --> MX33[Minimax AI33<br/>Tối đa 30 luồng/key<br/>10 request/phút]
    
    AI84 --> EL84[ElevenLabs AI84<br/>Tối đa 30 luồng/key<br/>5 request/phút]
    AI84 --> MX84[Minimax AI84<br/>Tối đa 30 luồng/key<br/>5 request/phút]
    
    EL33 --> APIPool33[Nhóm API Key AI33<br/>Nhiều Key chạy song song]
    MX33 --> APIPool33
    
    EL84 --> APIPool84[Nhóm API Key AI84<br/>Nhiều Key chạy song song]
    MX84 --> APIPool84
    
    SM -->|Giám sát| ConfigMgr[Quản lý Cấu hình<br/>4 Trường hợp + 3 Bảo trì]
    
    ConfigMgr -.->|Điều khiển| AI33
    ConfigMgr -.->|Điều khiển| AI84
```

## 2. Sơ đồ Phân phối Task (Chia việc)

```mermaid
graph LR
    subgraph "Hàng đợi Task"
        T1[Task 1]
        T2[Task 2]
        T3[Task 3]
        T4[Task 4]
        T5[Task 5]
        T6[Task 6]
        T7[Task 7]
        T8[Task 8]
        T9[Task 9]
        T10[Task 10]
    end
    
    subgraph "Bộ cân bằng tải"
        LB[Phân phối Task<br/>AI33: 70%<br/>AI84: 30%]
    end
    
    subgraph "AI33 Xử lý - 7 Task"
        direction TB
        AI33_Q[Hàng đợi AI33]
        AI33_D[Chia cho các API Key]
        
        K1[API Key 1<br/>Task 1,2]
        K2[API Key 2<br/>Task 3,4]
        K3[API Key 3<br/>Task 5,6]
        K4[API Key 4<br/>Task 7]
        
        AI33_Q --> AI33_D
        AI33_D --> K1
        AI33_D --> K2
        AI33_D --> K3
        AI33_D --> K4
    end
    
    subgraph "AI84 Xử lý - 3 Task"
        direction TB
        AI84_Q[Hàng đợi AI84]
        AI84_D[Chia cho các API Key]
        
        K5[API Key 1<br/>Task 8,9]
        K6[API Key 2<br/>Task 10]
        
        AI84_Q --> AI84_D
        AI84_D --> K5
        AI84_D --> K6
    end
    
    T1 --> LB
    T2 --> LB
    T3 --> LB
    T4 --> LB
    T5 --> LB
    T6 --> LB
    T7 --> LB
    T8 --> LB
    T9 --> LB
    T10 --> LB
    
    LB -->|70% - 7 task| AI33_Q
    LB -->|30% - 3 task| AI84_Q
```

## 3. Sơ đồ Cấu hình 4 Trường hợp + 3 Bảo trì

```mermaid
stateDiagram-v2
    [*] --> KiemTraCauHinh: Người dùng gửi yêu cầu
    
    KiemTraCauHinh --> TruongHop1: AI33 EL+MX BẬT<br/>AI84 Tất cả TẮT
    KiemTraCauHinh --> TruongHop2: AI33 Tất cả TẮT<br/>AI84 EL+MX BẬT
    KiemTraCauHinh --> TruongHop3: AI33 MX BẬT<br/>AI84 EL BẬT
    KiemTraCauHinh --> TruongHop4: AI33 EL BẬT<br/>AI84 MX BẬT
    
    KiemTraCauHinh --> BaoTri1: Bảo trì EL<br/>AI33 + AI84
    KiemTraCauHinh --> BaoTri2: Bảo trì MX<br/>AI33 + AI84
    KiemTraCauHinh --> BaoTri3: Bảo trì toàn bộ<br/>Server KingCong
    
    TruongHop1 --> XuLy1[Dùng AI33<br/>ElevenLabs + Minimax]
    TruongHop2 --> XuLy2[Dùng AI84<br/>ElevenLabs + Minimax]
    TruongHop3 --> XuLy3[Dùng AI33 Minimax<br/>+ AI84 ElevenLabs]
    TruongHop4 --> XuLy4[Dùng AI33 ElevenLabs<br/>+ AI84 Minimax]
    
    BaoTri1 --> ThongBaoLoi1[Thông báo lỗi:<br/>ElevenLabs đang bảo trì]
    BaoTri2 --> ThongBaoLoi2[Thông báo lỗi:<br/>Minimax đang bảo trì]
    BaoTri3 --> ThongBaoLoi3[Thông báo lỗi:<br/>Server KingCong<br/>đang bảo trì]
    
    XuLy1 --> TruCredits[Trừ Credits]
    XuLy2 --> TruCredits
    XuLy3 --> TruCredits
    XuLy4 --> TruCredits
    
    TruCredits --> [*]
    ThongBaoLoi1 --> [*]
    ThongBaoLoi2 --> [*]
    ThongBaoLoi3 --> [*]
```

## 4. Sơ đồ Quản lý Credits

```mermaid
graph TB
    subgraph "Hệ thống Credits"
        NguoiDung[Tài khoản người dùng] --> GoiCredits[Các gói Credits]
        
        GoiCredits --> Goi1[Gói 1: 5$ = 1,000,000 credits]
        GoiCredits --> Goi2[Gói 2-7: ...]
        GoiCredits --> Goi8[Gói 8: 40$ = 10,800,000 credits]
        
        Goi1 --> ViTien[Ví người dùng<br/>KingCong Credits]
        Goi2 --> ViTien
        Goi8 --> ViTien
    end
    
    subgraph "Theo dõi sử dụng"
        ViTien -->|Kiểm tra số dư| TaskXuLy[Xử lý Task]
        TaskXuLy -->|Tính chi phí| TinhToan[Tính toán chi phí]
        
        TinhToan --> ChiPhiAI33[Giá AI33<br/>theo ElevenLabs/Minimax]
        TinhToan --> ChiPhiAI84[Giá AI84<br/>theo ElevenLabs/Minimax]
        
        ChiPhiAI33 --> TruTien[Trừ Credits]
        ChiPhiAI84 --> TruTien
        
        TruTien --> CapNhatDB[(Cập nhật CSDL)]
        CapNhatDB --> ViTien
    end
    
    subgraph "Tính giá theo dịch vụ"
        DichVu[Dịch vụ sử dụng] --> KiemTra{Cái nào đang bật?}
        KiemTra -->|AI33 EL| Gia1[Giá AI33 ElevenLabs]
        KiemTra -->|AI33 MX| Gia2[Giá AI33 Minimax]
        KiemTra -->|AI84 EL| Gia3[Giá AI84 ElevenLabs]
        KiemTra -->|AI84 MX| Gia4[Giá AI84 Minimax]
        
        Gia1 --> ChiPhiCuoi[Tính chi phí cuối cùng]
        Gia2 --> ChiPhiCuoi
        Gia3 --> ChiPhiCuoi
        Gia4 --> ChiPhiCuoi
    end
```

## 5. Sơ đồ Luồng xử lý Yêu cầu

```mermaid
sequenceDiagram
    participant U as Người dùng
    participant FE as Giao diện
    participant BE as Backend
    participant CM as Quản lý Credits
    participant TM as Quản lý Task
    participant AI33 as Server AI33
    participant AI84 as Server AI84
    participant DB as Cơ sở dữ liệu
    
    U->>FE: Gửi yêu cầu TTS/Clone Voice
    FE->>BE: Chuyển yêu cầu
    
    BE->>CM: Kiểm tra số dư Credits
    CM->>DB: Truy vấn Credits người dùng
    DB-->>CM: Trả về số dư
    
    alt Credits không đủ
        CM-->>BE: Số dư không đủ
        BE-->>FE: Lỗi: Cần mua thêm credits
        FE-->>U: Hiển thị thông báo lỗi
    else Credits đủ
        CM-->>BE: Số dư OK
        
        BE->>TM: Tạo các Task
        TM->>TM: Chia thành 10 task
        
        par 70% cho AI33 (7 task)
            TM->>AI33: Giao Task 1-7
            AI33->>AI33: Chia cho các API Key
            AI33->>AI33: Xử lý với giới hạn<br/>(30 luồng/key, 10 req/phút)
            AI33-->>TM: Trả kết quả
        and 30% cho AI84 (3 task)
            TM->>AI84: Giao Task 8-10
            AI84->>AI84: Chia cho các API Key
            AI84->>AI84: Xử lý với giới hạn<br/>(30 luồng/key, 5 req/phút)
            AI84-->>TM: Trả kết quả
        end
        
        TM->>CM: Tính tổng chi phí
        CM->>DB: Trừ Credits
        DB-->>CM: Số dư đã cập nhật
        
        TM-->>BE: Tất cả Task hoàn thành
        BE-->>FE: Trả kết quả
        FE-->>U: Hiển thị kết quả + Credits còn lại
    end
```

## 6. Cấu trúc Giao diện (Tab)

```
📁 AI Tools Server KingCong
như FE server 1 server 2
```

## 7. Cấu trúc Cơ sở dữ liệu

```sql
-- Bảng Credits người dùng
CREATE TABLE kingcong_credits (
    user_id INT PRIMARY KEY,
    balance BIGINT NOT NULL DEFAULT 0,  -- Số dư hiện tại
    total_purchased BIGINT DEFAULT 0,   -- Tổng đã mua
    total_spent BIGINT DEFAULT 0,       -- Tổng đã tiêu
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Bảng Giao dịch
CREATE TABLE kingcong_transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    type ENUM('purchase', 'usage', 'refund'),  -- mua, dùng, hoàn
    amount BIGINT,
    service_used ENUM('ai33_elevenlabs', 'ai33_minimax', 'ai84_elevenlabs', 'ai84_minimax'),
    task_id VARCHAR(255),
    balance_after BIGINT,  -- Số dư sau giao dịch
    created_at TIMESTAMP
);

-- Bảng Cấu hình hệ thống
CREATE TABLE kingcong_config (
    id INT PRIMARY KEY,
    ai33_elevenlabs_active BOOLEAN DEFAULT TRUE,   -- AI33 EL bật/tắt
    ai33_minimax_active BOOLEAN DEFAULT TRUE,      -- AI33 MX bật/tắt
    ai84_elevenlabs_active BOOLEAN DEFAULT FALSE,  -- AI84 EL bật/tắt
    ai84_minimax_active BOOLEAN DEFAULT FALSE,     -- AI84 MX bật/tắt
    maintenance_mode ENUM('none', 'elevenlabs', 'minimax', 'full') DEFAULT 'none',
    task_split_ratio_ai33 INT DEFAULT 70,  -- Tỷ lệ % cho AI33
    task_split_ratio_ai84 INT DEFAULT 30,  -- Tỷ lệ % cho AI84
    updated_at TIMESTAMP
);

-- Bảng Nhóm API Keys
CREATE TABLE kingcong_api_keys (
    id INT AUTO_INCREMENT PRIMARY KEY,
    server ENUM('ai33', 'ai84'),
    service ENUM('elevenlabs', 'minimax'),
    api_key VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,        -- Key có đang hoạt động
    current_usage INT DEFAULT 0,           -- Số luồng đang dùng
    max_threads INT DEFAULT 30,            -- Tối đa luồng
    rate_limit_per_min INT,                -- Giới hạn request/phút
    last_used TIMESTAMP                    -- Lần dùng cuối
);

-- Bảng Hàng đợi Task
CREATE TABLE kingcong_tasks (
    id VARCHAR(255) PRIMARY KEY,
    user_id INT,
    status ENUM('pending', 'processing', 'completed', 'failed'),  -- chờ, đang xử lý, hoàn thành, lỗi
    assigned_to ENUM('ai33', 'ai84'),      -- Giao cho server nào
    service ENUM('elevenlabs', 'minimax'), -- Dùng dịch vụ nào
    api_key_id INT,                        -- Key nào đang xử lý
    input_data TEXT,                       -- Dữ liệu đầu vào
    output_data TEXT,                      -- Kết quả đầu ra
    credits_cost BIGINT,                   -- Chi phí credits
    created_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- Bảng Bảng giá dịch vụ
CREATE TABLE kingcong_pricing (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(100),             -- Tên dịch vụ
    server ENUM('ai33', 'ai84'),
    service_type ENUM('elevenlabs', 'minimax'),
    credits_per_character DECIMAL(10, 4),  -- Credits/ký tự
    credits_per_second DECIMAL(10, 4),     -- Credits/giây âm thanh
    is_active BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMP
);
```

## 8. Logic Backend (Code giả)

```python
class BoPhanPhoBKingCong:
    def __init__(self):
        self.ty_le_ai33 = 0.70  # 70%
        self.ty_le_ai84 = 0.30  # 30%
        self.cau_hinh = self.tai_cau_hinh()
    
    def xu_ly_yeu_cau(self, user_id, du_lieu_yeu_cau):
        # 1. Kiểm tra credits
        if not self.kiem_tra_credits(user_id, du_lieu_yeu_cau):
            return {"loi": "Credits không đủ, vui lòng nạp thêm"}
        
        # 2. Kiểm tra chế độ bảo trì
        if self.cau_hinh.che_do_bao_tri == 'full':
            return {"loi": "Server đang bảo trì, vui lòng thử lại sau"}
        
        # 3. Xác định dịch vụ đang hoạt động
        dich_vu_hoat_dong = self.lay_dich_vu_hoat_dong()
        if not dich_vu_hoat_dong:
            return {"loi": "Không có dịch vụ nào khả dụng"}
        
        # 4. Tạo các task
        cac_task = self.tao_task(du_lieu_yeu_cau)
        
        # 5. Phân phối task
        so_task_ai33 = int(len(cac_task) * self.ty_le_ai33)
        task_ai33 = cac_task[:so_task_ai33]
        task_ai84 = cac_task[so_task_ai33:]
        
        # 6. Xử lý với nhóm API key
        ket_qua = []
        
        if self.cau_hinh.ai33_hoat_dong():
            ket_qua_ai33 = self.xu_ly_ai33(task_ai33, dich_vu_hoat_dong)
            ket_qua.extend(ket_qua_ai33)
        
        if self.cau_hinh.ai84_hoat_dong():
            ket_qua_ai84 = self.xu_ly_ai84(task_ai84, dich_vu_hoat_dong)
            ket_qua.extend(ket_qua_ai84)
        
        # 7. Tính và trừ credits
        tong_chi_phi = self.tinh_chi_phi(ket_qua)
        self.tru_credits(user_id, tong_chi_phi)
        
        return {
            "thanh_cong": True, 
            "ket_qua": ket_qua,
            "credits_da_dung": tong_chi_phi
        }
    
    def xu_ly_ai33(self, cac_task, dich_vu):
        """Xử lý task trên AI33"""
        cac_api_key = self.lay_key_kha_dung('ai33', dich_vu)
        return self.phan_bo_cho_key(
            cac_task, 
            cac_api_key, 
            max_luong=30,      # Tối đa 30 luồng/key
            gioi_han_req=10    # 10 request/phút
        )
    
    def xu_ly_ai84(self, cac_task, dich_vu):
        """Xử lý task trên AI84"""
        cac_api_key = self.lay_key_kha_dung('ai84', dich_vu)
        return self.phan_bo_cho_key(
            cac_task,
            cac_api_key,
            max_luong=30,      # Tối đa 30 luồng/key
            gioi_han_req=5     # 5 request/phút
        )
    
    def phan_bo_cho_key(self, cac_task, cac_api_key, max_luong, gioi_han_req):
        """Phân bổ task cho các API key theo vòng tròn"""
        ket_qua = []
        
        for i, task in enumerate(cac_task):
            # Chọn key theo vòng tròn
            key = cac_api_key[i % len(cac_api_key)]
            
            # Kiểm tra giới hạn request
            if self.kiem_tra_gioi_han(key, gioi_han_req):
                ket_qua_task = self.thuc_thi_task(task, key)
                ket_qua.append(ket_qua_task)
            else:
                # Đưa vào hàng đợi hoặc dùng key khác
                ket_qua_task = self.xu_ly_khi_vuot_gioi_han(task, cac_api_key)
                ket_qua.append(ket_qua_task)
        
        return ket_qua
    
    def tinh_chi_phi(self, ket_qua):
        """Tính tổng chi phí credits"""
        tong = 0
        for kq in ket_qua:
            if kq['dich_vu'] == 'ai33_elevenlabs':
                tong += kq['so_ky_tu'] * self.gia_ai33_el
            elif kq['dich_vu'] == 'ai33_minimax':
                tong += kq['so_ky_tu'] * self.gia_ai33_mx
            elif kq['dich_vu'] == 'ai84_elevenlabs':
                tong += kq['so_ky_tu'] * self.gia_ai84_el
            elif kq['dich_vu'] == 'ai84_minimax':
                tong += kq['so_ky_tu'] * self.gia_ai84_mx
        return tong
    
    def tru_credits(self, user_id, so_tien):
        """Trừ credits từ tài khoản người dùng"""
        # Cập nhật database
        db.cap_nhat(
            "UPDATE kingcong_credits SET balance = balance - %s WHERE user_id = %s",
            (so_tien, user_id)
        )
        
        # Ghi log giao dịch
        db.them(
            "INSERT INTO kingcong_transactions (user_id, type, amount) VALUES (%s, 'usage', %s)",
            (user_id, so_tien)
        )
```

## 9. Các trường hợp cụ thể

### Trường hợp 1: AI33 ElevenLabs BẬT, AI33 Minimax BẬT, AI84 Tất cả TẮT
```
Luồng xử lý:
1. Nhận 10 task từ người dùng
2. Phân bổ: 7 task → AI33, 3 task → AI84 (nhưng AI84 tắt)
3. AI33 xử lý cả 10 task
4. AI33 chia 10 task cho nhiều API key ElevenLabs và Minimax
5. Tính credits theo giá AI33
```

### Trường hợp 2: AI33 Tất cả TẮT, AI84 ElevenLabs BẬT, AI84 Minimax BẬT
```
Luồng xử lý:
1. Nhận 10 task từ người dùng
2. Phân bổ: 7 task → AI33 (nhưng tắt), 3 task → AI84
3. AI84 xử lý cả 10 task
4. AI84 chia 10 task cho nhiều API key ElevenLabs và Minimax
5. Tính credits theo giá AI84
```

### Trường hợp 3: AI33 Minimax BẬT, AI84 ElevenLabs BẬT
```
Luồng xử lý:
1. Nhận 10 task từ người dùng
2. Phân loại task: Task nào cần ElevenLabs? Task nào cần Minimax?
3. Task ElevenLabs → AI84 xử lý
4. Task Minimax → AI33 xử lý
5. Tính credits hỗn hợp (AI33 cho Minimax, AI84 cho ElevenLabs)
```

### Trường hợp 4: AI33 ElevenLabs BẬT, AI84 Minimax BẬT
```
Luồng xử lý:
1. Nhận 10 task từ người dùng
2. Phân loại task: Task nào cần ElevenLabs? Task nào cần Minimax?
3. Task ElevenLabs → AI33 xử lý
4. Task Minimax → AI84 xử lý
5. Tính credits hỗn hợp (AI33 cho ElevenLabs, AI84 cho Minimax)
```

---

## 10. Checklist triển khai

### Backend
- [ ] Tạo bảng cơ sở dữ liệu (7 bảng)
- [ ] Xây dựng API endpoint cho mua credits
- [ ] Xây dựng API endpoint cho Text-to-Speech
- [ ] Xây dựng API endpoint cho Voice Cloning
- [ ] Viết logic phân phối task (70-30)
- [ ] Viết logic quản lý API key pool
- [ ] Viết logic kiểm tra rate limit
- [ ] Viết logic tính toán credits
- [ ] Viết logic xử lý 4 trường hợp + 3 bảo trì
- [ ] Viết cron job theo dõi trạng thái server

### Frontend
- [ ] Tạo tab "AI Tools Server KingCong"
- [ ] Tạo tab con "Mua Credits" với 8 gói
- [ ] Tạo tab con "Text To Speech"
- [ ] Tạo tab con "Voice Cloning"
- [ ] Hiển thị thanh trạng thái (Credits, AI33, AI84)
- [ ] Hiển thị thông báo bảo trì
- [ ] Tích hợp thanh toán
- [ ] Hiển thị lịch sử giao dịch

### Kiểm thử
- [ ] Test phân phối 70-30
- [ ] Test rate limit AI33 (10 req/phút)
- [ ] Test rate limit AI84 (5 req/phút)
- [ ] Test cả 4 trường hợp hoạt động
- [ ] Test cả 3 chế độ bảo trì
- [ ] Test tính credits chính xác
- [ ] Test xử lý khi credits không đủ
- [ ] Load test với 100+ task đồng thời

### Admin Panel (Khuyến nghị)
- [ ] Bật/tắt AI33 ElevenLabs
- [ ] Bật/tắt AI33 Minimax
- [ ] Bật/tắt AI84 ElevenLabs
- [ ] Bật/tắt AI84 Minimax
- [ ] Chuyển chế độ bảo trì
- [ ] Điều chỉnh tỷ lệ phân phối (70-30)
- [ ] Thêm/xóa API key
- [ ] Xem thống kê sử dụng
- [ ] Xem logs lỗi

---

## Tổng kết

### Ưu điểm của kiến trúc này:
✅ **Linh hoạt**: 4 trường hợp + 3 bảo trì đáp ứng mọi tình huống  
✅ **Tối ưu tốc độ**: Chia task cho nhiều API key chạy song song  
✅ **Cân bằng tải**: 70-30 có thể điều chỉnh theo nhu cầu  
✅ **Tiết kiệm chi phí**: Tự quản lý credits, không phụ thuộc AI33/AI84  
✅ **Dễ mở rộng**: Thêm server AI mới chỉ cần thêm cấu hình  
