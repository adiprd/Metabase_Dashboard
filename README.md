# Metabase - Deployment & Dashboard Guide

## 📋 Daftar Isi
- [Overview Metabase](#overview-metabase)
- [Deployment Metabase](#deployment-metabase)
- [Konfigurasi Database](#konfigurasi-database)
- [Pembuatan Dashboard](#pembuatan-dashboard)
- [Best Practices](#best-practices)
- [Maintenance](#maintenance)

## 🚀 Overview Metabase

### Apa itu Metabase?
Metabase adalah open-source business intelligence tool yang memungkinkan:
- **Visualisasi Data** yang user-friendly
- **Query Building** tanpa kode SQL
- **Dashboard Interaktif** dengan filter real-time
- **Sharing & Embedding** reports
- **Scheduled Alerts** dan subscriptions

### Keunggulan
- ✅ Mudah digunakan oleh non-technical users
- ✅ Open-source dan gratis
- ✅ Integrasi dengan berbagai database
- ✅ Deployment yang sederhana

## 🛠 Deployment Metabase

### 1. Deployment dengan Docker (Recommended)

```dockerfile
# docker-compose.yml
version: '3.8'
services:
  metabase:
    image: metabase/metabase:latest
    container_name: metabase
    hostname: metabase
    ports:
      - "3000:3000"
    environment:
      - MB_DB_TYPE=postgres
      - MB_DB_DBNAME=metabase
      - MB_DB_PORT=5432
      - MB_DB_HOST=postgresql-host
      - MB_DB_USER=metabase_user
      - MB_DB_PASS=your_secure_password
    volumes:
      - /app/metabase/data:/metabase-data
    restart: unless-stopped
    networks:
      - metabase-network

networks:
  metabase-network:
    driver: bridge
```

### 2. Deployment dengan Kubernetes

```yaml
# metabase-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metabase
  namespace: analytics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: metabase
  template:
    metadata:
      labels:
        app: metabase
    spec:
      containers:
      - name: metabase
        image: metabase/metabase:latest
        ports:
        - containerPort: 3000
        env:
        - name: MB_DB_TYPE
          value: "postgres"
        - name: MB_DB_DBNAME
          value: "metabase"
        - name: MB_DB_HOST
          value: "postgresql-service"
        - name: MB_DB_USER
          valueFrom:
            secretKeyRef:
              name: metabase-secret
              key: username
        - name: MB_DB_PASS
          valueFrom:
            secretKeyRef:
              name: metabase-secret
              key: password
        volumeMounts:
        - name: metabase-data
          mountPath: /metabase-data
      volumes:
      - name: metabase-data
        persistentVolumeClaim:
          claimName: metabase-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: metabase-service
  namespace: analytics
spec:
  selector:
    app: metabase
  ports:
  - port: 80
    targetPort: 3000
  type: LoadBalancer
```

### 3. Deployment Manual (JAR File)

```bash
# Download Metabase
wget https://downloads.metabase.com/v0.46.3/metabase.jar

# Run Metabase
java -jar metabase.jar

# Dengan custom configuration
java -DMB_DB_TYPE=postgres \
     -DMB_DB_CONNECTION_URI="jdbc:postgresql://localhost:5432/metabase?user=mbuser&password=mbpass" \
     -jar metabase.jar
```

## 🔗 Konfigurasi Database

### 1. Menghubungkan PostgreSQL Database

```sql
-- Buat user khusus untuk Metabase
CREATE USER metabase_user WITH PASSWORD 'secure_password';

-- Berikan permissions
GRANT CONNECT ON DATABASE your_database TO metabase_user;
GRANT USAGE ON SCHEMA mb TO metabase_user;
GRANT SELECT ON ALL TABLES IN SCHEMA mb TO metabase_user;

-- Untuk tabel baru yang akan dibuat
ALTER DEFAULT PRIVILEGES IN SCHEMA mb 
GRANT SELECT ON TABLES TO metabase_user;
```

### 2. Connection String di Metabase

```
Jdbc URL: jdbc:postgresql://your-postgres-host:5432/your_database
Username: metabase_user
Password: secure_password
```

### 3. Konfigurasi Schema Access

```sql
-- Berikan akses ke schema mb
GRANT USAGE ON SCHEMA mb TO metabase_user;
GRANT SELECT ON ALL TABLES IN SCHEMA mb TO metabase_user;
```

## 📊 Pembuatan Dashboard

### 1. Persiapan Data Model

#### a. Review Tabel yang Tersedia
```sql
-- Cek struktur tabel
SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_schema = 'mb' 
AND table_name = 'data_transaksi_sparepart';
```

#### b. Buat Views untuk Simplifikasi
```sql
-- View untuk summary harian
CREATE OR REPLACE VIEW mb.vw_summary_transaksi_harian AS
SELECT 
    tgl_invoice,
    cabang,
    COUNT(DISTINCT no_invoice) as total_invoice,
    SUM(total_harga) as total_penjualan,
    SUM(qty) as total_qty,
    COUNT(DISTINCT nama_customer) as total_customer
FROM mb.data_transaksi_sparepart
GROUP BY tgl_invoice, cabang;

-- View untuk performance produk
CREATE OR REPLACE VIEW mb.vw_top_produk AS
SELECT 
    kode_barang,
    nama_barang,
    tipe_sparepart,
    SUM(qty) as total_terjual,
    SUM(total_harga) as total_revenue,
    COUNT(DISTINCT no_invoice) as frekuensi_penjualan
FROM mb.data_transaksi_sparepart
GROUP BY kode_barang, nama_barang, tipe_sparepart;
```

### 2. Membuat Pertanyaan (Questions)

#### a. Total Penjualan per Bulan
- **Tipe**: Line Chart
- **Data**: 
  - X-axis: Month(tgl_invoice)
  - Y-axis: Sum(total_harga)
- **Filter**: 
  - Tahun berjalan
  - Cabang tertentu (optional)

#### b. Top 10 Produk Terlaris
- **Tipe**: Bar Chart
- **Data**:
  - X-axis: nama_barang
  - Y-axis: sum(qty)
- **Filter**:
  - Rentang tanggal
  - Tipe sparepart

#### c. Performance per Cabang
- **Tipe**: Pie Chart
- **Data**:
  - Kategori: cabang
  - Metric: sum(total_harga)
- **Filter**:
  - Bulan berjalan

### 3. Membuat Dashboard Interaktif

#### Dashboard: "Performance Penjualan Sparepart"

**Layout:**
```
┌─────────────────┬─────────────────┐
│  Total Revenue  │ Top 5 Cabang    │
│  Current Month  │ by Sales        │
├─────────────────┼─────────────────┤
│  Sales Trend    │ Product         │
│  (6 Months)     │ Performance     │
├─────────────────┼─────────────────┤
│  Customer       │ Monthly         │
│  Distribution   │ Growth          │
└─────────────────┴─────────────────┘
```

**Komponen Dashboard:**

1. **KPI Cards**
   - Total Revenue Bulan Ini
   - Growth vs Bulan Lalu
   - Total Transaction
   - Average Order Value

2. **Charts**
   - Sales Trend (Line Chart)
   - Revenue by Branch (Bar Chart)
   - Product Category Performance (Pie Chart)
   - Customer Distribution (Map)

### 4. Filter Dashboard

```sql
-- Contoh filter untuk dashboard
-- 1. Filter Tanggal
WHERE tgl_invoice BETWEEN {{start_date}} AND {{end_date}}

-- 2. Filter Cabang
WHERE cabang IN ({{selected_branches}})

-- 3. Filter Tipe Sparepart
WHERE tipe_sparepart = {{sparepart_type}}
```

## 🎯 Best Practices

### 1. Data Modeling
```sql
-- Buat index untuk performa query
CREATE INDEX idx_transaksi_tgl ON mb.data_transaksi_sparepart(tgl_invoice);
CREATE INDEX idx_transaksi_cabang ON mb.data_transaksi_sparepart(cabang);
CREATE INDEX idx_transaksi_produk ON mb.data_transaksi_sparepart(kode_barang);
```

### 2. Query Optimization
- Gunakan aggregated tables untuk data yang sering diakses
- Implementasi materialized views untuk complex queries
- Batasi data yang ditampilkan dengan filter yang tepat

### 3. Dashboard Design
- Group related questions dalam section
- Gunakan consistent color scheme
- Tambahkan text boxes untuk penjelasan
- Implementasi dashboard-level filters

### 4. Security
```sql
-- Row Level Security (jika diperlukan)
CREATE POLICY sales_data_policy ON mb.data_transaksi_sparepart
FOR SELECT TO metabase_user
USING (cabang = current_setting('app.current_branch'));
```

## 🔧 Maintenance

### 1. Monitoring Performance
```sql
-- Query performance monitoring
SELECT 
    query,
    execution_time,
    result_rows
FROM metabase_query_execution
ORDER BY execution_time DESC
LIMIT 10;
```

### 2. Backup Configuration
```bash
# Backup Metabase database
pg_dump -h localhost -U metabase_user metabase > metabase_backup_$(date +%Y%m%d).sql

# Backup application data
tar -czf metabase_data_$(date +%Y%m%d).tar.gz /app/metabase/data
```

### 3. Update Strategy
```bash
# Untuk Docker deployment
docker-compose pull
docker-compose up -d

# Untuk manual deployment
wget https://downloads.metabase.com/v0.46.3/metabase.jar
java -jar metabase.jar
```

### 4. Health Check
```bash
# Health check endpoint
curl http://localhost:3000/api/health

# Expected response
{"status":"ok"}
```

## 📈 Advanced Features

### 1. Scheduled Reports
- Email reports harian/mingguan/bulanan
- Automated PDF exports
- Slack notifications

### 2. Embedded Analytics
```javascript
// Embed dashboard di web application
<iframe
  src="http://metabase.example.com/embed/dashboard/{{token}}"
  width="100%"
  height="800"
  allowtransparency
></iframe>
```

### 3. Custom CSS (White-labeling)
```css
/* Custom branding */
.navbar-brand {
  background-image: url('/your-logo.png');
}
```

## 🚨 Troubleshooting

### Common Issues & Solutions

1. **Connection Timeout**
   ```bash
   # Increase timeout
   MB_JETTY_TIMEOUT=120
   ```

2. **Memory Issues**
   ```bash
   # Increase heap size
   java -Xmx2g -jar metabase.jar
   ```

3. **Slow Queries**
   - Optimize database indexes
   - Use aggregated tables
   - Implement query caching

4. **Permission Denied**
   ```sql
   -- Grant necessary permissions
   GRANT SELECT ON schema.table TO metabase_user;
   ```

## 📞 Support

### Resources
- [Metabase Documentation](https://www.metabase.com/docs/latest/)
- [Community Forum](https://discourse.metabase.com/)
- [GitHub Issues](https://github.com/metabase/metabase/issues)

### Monitoring Tools
- Metabase built-in analytics
- Database query performance monitoring
- Application performance monitoring (APM)

Dengan guide ini, Anda dapat melakukan deployment Metabase yang robust dan membuat dashboard yang powerful untuk analisis data transaksi sparepart.
