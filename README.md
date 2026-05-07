# รายงานโครงการ: ระบบวิเคราะห์พฤติกรรมเครดิตอัตโนมัติบนระบบคลาวด์
**Project Title: Automated Cloud-Based Credit Behavioral Scoring Pipeline**

## 1. บทนำและสถานการณ์ (Executive Summary)
ในอุตสาหกรรมการเงินยุคปัจจุบัน การพิจารณาสินเชื่อไม่ได้จำกัดอยู่เพียงข้อมูลรายได้คงที่ (Static Data) เท่านั้น โครงการนี้จึงถูกพัฒนาขึ้นเพื่อสร้าง **Automated Data Pipeline** ในการดึงข้อมูลพฤติกรรมการใช้จ่ายรายวัน (Alternative Data) มาเปลี่ยนเป็นคะแนนเครดิต (**Behavioral Score**) โดยเน้นการประมวลผลแบบ Real-time และความสามารถในการขยายตัว (Scalability) เพื่อรองรับฐานลูกค้าที่ขยายตัวอย่างต่อเนื่องบนระบบสถาปัตยกรรม Serverless ของ AWS

## 2. ปัญหาและอุปสรรค (Challenges & Problem Statement)
* **Data Quality Issue:** ข้อมูลธุรกรรมมีความหลากหลาย เช่น รูปแบบวันที่ที่ผิดเพี้ยน (Dirty Formats), ข้อมูลซ้ำซ้อน (Duplicates) และค่าว่าง (Null)
* **Inflexible Scoring:** สูตรการให้คะแนนในระยะแรกไม่สามารถแยกแยะกลุ่มลูกค้าได้ชัดเจน (Score Clustering) ทำให้เกิดปัญหาคะแนนเกาะกลุ่มสูงเกินไป (Skewed towards 100)
* **Automation Gap:** การเชื่อมต่อจาก Local Environment ไปยัง Cloud ต้องมีความมั่นคง ปลอดภัย และมีระบบ Logging ที่ตรวจสอบย้อนหลังได้
* **Operational Latency:** ความแตกต่างของ Time-zone (UTC vs ICT) ส่งผลต่อรอบการประมวลผลสรุปยอดประจำวัน

## 3. สถาปัตยกรรมระบบ (System Architecture)


ระบบถูกออกแบบโดยแบ่งเป็น 4 เลเยอร์หลักตามมาตรฐาน Data Engineering:

1.  **Ingestion Layer:** * ใช้ Python Script (`sender.py`) ส่งข้อมูล JSON ผ่าน **AWS API Gateway**
    * บันทึกประวัติการส่งข้อมูลลงในไฟล์ `transfer_history.csv` เพื่อการตรวจสอบฝั่งต้นทาง
2.  **Processing Layer (Bronze to Silver):**
    * **Data Cleaning:** ใช้ AWS Lambda จัดการกำจัด Duplicates, Standardize วันที่ให้เป็น ISO-8601 และกรองค่า Null
    * **Storage:** เก็บข้อมูลที่พร้อมใช้งานลงใน **Amazon S3 (Processed Bucket)**
3.  **Analytics & Integration Layer (Silver to Gold):**
    * **Orchestration:** ตั้งค่า **EventBridge (Cron Job)** ให้ระบบเริ่มทำงานทุกเที่ยงคืน 5 นาที (เวลาไทย)
    * **Feature Engineering:** คำนวณ `essential_spending_ratio` และ `behavioral_score`
    * **Data Mart:** บันทึกผลลัพธ์ลงใน **Amazon RDS (MySQL)** ทั้งตารางปัจจุบัน (Mart) และตารางประวัติ (History)
4.  **Visualization Layer:**
    * เชื่อมต่อฐานข้อมูลเข้ากับ **Looker Studio** เพื่อสร้าง Dashboard สำหรับติดตามความเสี่ยงและพฤติกรรมลูกค้า

## 4. กลยุทธ์การคำนวณคะแนน (Feature Engineering Logic)
เพื่อให้คะแนนสะท้อนพฤติกรรมที่แท้จริง ระบบใช้การถ่วงน้ำหนัก (Weighting) 4 ปัจจัยสำคัญ:
* **Total Spending (40%):** ปริมาณเงินหมุนเวียน (Cap ที่ 500,000 THB)
* **Category Diversity (30%):** ความหลากหลายของไลฟ์สไตล์ (วัดจาก Distinct Categories)
* **Transaction Frequency (20%):** ความสม่ำเสมอและความถี่ในการใช้งาน
* **Purchase Power (10%):** กำลังซื้อเฉลี่ยต่อธุรกรรม (Average Ticket Size)

## 5. ผลลัพธ์และการวิเคราะห์ (Key Results)
* **Normal Distribution:** หลังจากปรับปรุง Scoring Logic พบว่าคะแนนลูกค้ามีการกระจายตัวที่สมดุลมากขึ้น ช่วยให้จำแนกกลุ่มลูกค้า High-risk และ Low-risk ได้อย่างมีประสิทธิภาพ
* **Efficiency:** ลดภาระงาน Manual ได้ 100% ตั้งแต่ขั้นตอนการนำเข้าข้อมูลจนถึงการออกรายงาน
* **Auditability:** มี Log ครบวงจรทั้งบนเครื่อง Local และบน Cloud (Data Load Logs)

## 6. ข้อเสนอแนะและงานในอนาคต (Future Work)
* **Machine Learning:** นำข้อมูลจาก Credit Feature Mart ไปพัฒนาต่อยอดเป็น Predictive Model สำหรับทำ Default Prediction
* **Fraud Detection:** เพิ่ม Layer การตรวจจับพฤติกรรมการใช้จ่ายที่ผิดปกติ (Anomaly Detection)
* **Real-time API:** เปิด Endpoint ให้ระบบภายนอกสามารถ Query คะแนนไปใช้พิจารณาสินเชื่อได้ทันที

