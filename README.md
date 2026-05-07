# รายงานโครงการฉบับสมบูรณ์: ระบบวิเคราะห์พฤติกรรมเครดิตอัตโนมัติบนสถาปัตยกรรมคลาวด์แบบ Serverless
**Project Title: End-to-End Automated Cloud-Based Credit Behavioral Scoring Pipeline**

---

## 1. บทนำและแรงจูงใจของโครงการ (Executive Summary & Motivation)
ในระบบการเงินแบบดั้งเดิม การพิจารณาวงเงินสินเชื่อมักอาศัยข้อมูลเชิงสถิติที่หยุดนิ่ง (Static Data) เช่น รายได้ประจำหรือประวัติค้างชำระ ซึ่งไม่สามารถสะท้อนศักยภาพที่แท้จริงของผู้กู้บางกลุ่มได้ โครงการนี้จึงมุ่งเน้นการสร้างระบบประมวลผลข้อมูลธุรกรรมรายวัน (Alternative Data) เพื่อวิเคราะห์พฤติกรรม (Behavioral Insights) และเปลี่ยนให้เป็นคะแนนที่จับต้องได้ โดยใช้เทคโนโลยี Cloud Computing ในรูปแบบ **Serverless Architecture** เพื่อให้ระบบมีความยืดหยุ่น ประหยัดค่าใช้จ่าย และทำงานได้โดยอัตโนมัติ (Fully Automated) ตั้งแต่ต้นน้ำจนถึงปลายน้ำ

## 2. การวิเคราะห์ปัญหาและข้อจำกัด (Problem Identification)
จากการทดลองเบื้องต้น พบปัญหาสำคัญ 3 ประการที่ต้องแก้ไขด้วยวิศวกรรมข้อมูล:
1.  **Data Quality & Integrity:** ข้อมูลที่รับมาจากระบบภายนอกมีความไม่เป็นระเบียบไม่สะอาดและมีปัญหาข้อมูลซ้ำ (Data Duplication) จากการส่งข้อมูลซ้ำในระบบเครือข่าย
2.  **Scoring Skewness:** สูตรการคำนวณแบบเดิม (Rule-based) ขาดความละเอียด ทำให้ลูกค้าส่วนใหญ่มีคะแนนสูงเกินจริง (Clustering at 100) ไม่สามารถนำมาจัดลำดับความเสี่ยง (Risk Ranking) ได้อย่างมีประสิทธิภาพ
3.  **Scalability & Observability:** การรันสคริปต์แบบ Manual ไม่สามารถรองรับการขยายตัวได้ และหากเกิดข้อผิดพลาดในขั้นตอนใดขั้นตอนหนึ่ง จะไม่สามารถทราบสาเหตุได้ทันทีหากขาดระบบ Logging ที่ดี

## 3. สถาปัตยกรรมทางเทคนิค (Detailed Technical Architecture)

### 3.1 Data Ingestion Layer (เลเยอร์การนำเข้าข้อมูล)
* **Source Side:** พัฒนา `sender.py` ให้ทำหน้าที่เป็น Ingester ที่มีระบบ **Local State Tracking** โดยการเก็บบันทึกประวัติลงใน `transfer_history.csv` เพื่อป้องกันการส่งไฟล์ซ้ำและตรวจสอบสถานะการส่งในระดับเครื่องต้นทาง
* **Cloud Gateway:** ใช้ **Amazon API Gateway** เป็นทางเข้าหลักเพื่อความปลอดภัย และส่งต่อให้ **AWS Lambda** บันทึกข้อมูลลงใน S3 โดยแยกโฟลเดอร์ตามวันเวลา (Partitioning) เพื่อความง่ายในการดึงข้อมูลย้อนหลัง (Data Archiving)

### 3.2 Data Processing & Transformation (เลเยอร์การประมวลผล)
* **Data Cleaning Pipeline:** Lambda Function จะทำการแปลงข้อมูลดิบ (Raw Data) ให้เป็นโครงสร้างมาตรฐาน (Silver Layer) โดยใช้ห้องสมุด **Pandas** ในการทำ:
    * **Deduplication:** กำจัดรายการธุรกรรมที่ซ้ำกันโดยใช้ `txn_id` เป็น Unique Key
    * **Standardization:** ปรับแต่งค่าสกุลเงิน (Currency Conversion) และรูปแบบวันที่ (Timestamp Normalization)
* **ETL Orchestration:** ใช้ **Amazon EventBridge** เป็นตัวควบคุมจังหวะเวลา (Scheduler) โดยตั้งค่าให้รันทุกวันในช่วงเวลาที่ Traffic ต่ำ (00:05 น.) เพื่อรวบรวมข้อมูลทั้งหมดของวันนั้นมาประมวลผลสรุปยอด

### 3.3 Database Design & Feature Mart (เลเยอร์การจัดเก็บข้อมูล)
ออกแบบระบบฐานข้อมูลบน **Amazon RDS (MySQL)** โดยแบ่งตารางเพื่อวัตถุประสงค์ที่ต่างกัน:
* **`credit_feature_mart`:** เก็บสถานะล่าสุดของลูกค้าแต่ละราย (Current State) เพื่อการ Query ที่รวดเร็ว
* **`credit_score_history`:** เก็บข้อมูลแบบ Time-series เพื่อใช้ดูแนวโน้มพฤติกรรม (Trend Analysis) และการเปลี่ยนแปลงของคะแนนรายวัน
* **`data_load_logs`:** เก็บ Audit Log ของการประมวลผลในระดับไฟล์ เพื่อใช้ในกระบวนการ Data Reconcile

## 4. การออกแบบฟีเจอร์และการให้คะแนน (Advanced Feature Engineering)
เราได้ออกแบบสูตรการให้คะแนนแบบ **Multi-factor Scoring Model** เพื่อกระจายตัวเลขคะแนนให้สะท้อนความเป็นจริง:

| Factor | Weight | Logic / Rationale |
| :--- | :--- | :--- |
| **Total Spending** | 40% | ปริมาณกระแสเงินสดหมุนเวียน (Cap 500,000 THB) |
| **Category Diversity** | 30% | ความหลากหลายของการใช้จ่าย สะท้อนถึงเสถียรภาพในการใช้ชีวิต |
| **Transaction Frequency** | 20% | ความถี่ในการใช้งานระบบ เพื่อวัดความสม่ำเสมอของพฤติกรรม |
| **Average Ticket Size** | 10% | กำลังซื้อต่อครั้ง เพื่อระบุระดับรายได้ (Wealth Segment) |

นอกจากนี้ยังมีการเพิ่มคอลัมน์ **`essential_spending_ratio`** เพื่อคำนวณสัดส่วนค่าใช้จ่ายจำเป็น (อาหาร, ค่าน้ำไฟ, ประกัน) ซึ่งเป็นตัวแปรสำคัญที่ใช้ประเมินความสามารถในการชำระหนี้ (Debt Serviceability)

## 5. การวิเคราะห์ผลลัพธ์และแดชบอร์ด (Results & Visualization)
จากการเชื่อมต่อ RDS เข้ากับ **Looker Studio** พบ Insight ที่สำคัญดังนี้:
* **Segmentation:** สามารถแบ่งลูกค้าออกเป็น 3 กลุ่มหลัก (Prime, Near-prime, Sub-prime) ตามการกระจายตัวของคะแนนแบบ **Normal Distribution**
* **Correlation:** พบความสัมพันธ์เชิงบวกระหว่างความหลากหลายของหมวดหมู่สินค้าและคะแนนความน่าเชื่อถือ
* **Monitoring:** ระบบสามารถรายงานจำนวนธุรกรรมที่ผ่านการประมวลผลสำเร็จเทียบกับที่ล้มเหลวได้แบบ Real-time ผ่าน Audit Logs

## 6. สรุปผลและทิศทางในอนาคต (Conclusion & Roadmap)
โครงการนี้พิสูจน์ให้เห็นว่าระบบ Serverless สามารถสร้าง Data Pipeline ที่ซับซ้อนได้โดยมีความซับซ้อนในการดูแลต่ำ (Low Operations)
* **Short-term:** เพิ่มระบบแจ้งเตือนผ่าน Line Notify หรือ Email เมื่อคะแนนลูกค้าบางรายตกลงอย่างผิดปกติ
* **Long-term:** พัฒนาไปสู่การใช้ **Machine Learning (AWS SageMaker)** เพื่อทำ Predictive Scoring โดยใช้ผลลัพธ์จาก Pipeline นี้เป็น Training Dataset เพื่อสร้างโมเดลที่มีความแม่นยำสูงกว่าระบบ Rule-based
