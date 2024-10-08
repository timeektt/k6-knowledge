# Test Types
การทำ Load testing แต่ละแบบ เราอาจทดสอบโดยใช้ script เดียวกันได้เลย ซึ่งแต่ละ script จะต่างกันแค่การกำหนดค่าใน options เท่านั้น

## Smoke Testing
เราใช้ Smoke Test เพื่อทดสอบว่าโดยทั่วไประบบของเรานั้นยังปกติดีไหม และยังใช้ทดสอบอีกว่า test script ของเรานั้นรันได้ถูกต้องเท่านั้น ดังนั้นจึงไม่จำเป็นต้องกำหนด VU หรือ duration ให้มากมายนักก็ได้

```
// 1 user looping for 1 minute
export let options = {
  vus: 1,
  duration: '1m',
  thresholds: {
    'http_req_duration': ['p(99)<1500'],
  }
};
```
## Load Testing
![image](https://github.com/user-attachments/assets/5bc0d2e7-f198-415e-b1a9-7f001aa3c0df)

ใช้ทดสอบ performance ของระบบว่าสามารถรองรับการใช้งานตามที่เราตั้งเป้าหมายไว้หรือไม่ เช่น เราคาดว่าระบบที่เราออกแบบมาต้องรองรับการใช้งานพร้อมๆ ได้ประมาณ 100 คน หากไม่สามารถรองรับการใช้งานในภาวะปกติได้ตามที่คาดไว้ เราจะได้ tune ระบบได้ก่อนปล่อยไปยัง production

```
export let options = {
  stages: [
    { duration: "5m", target: 100 }, 
    { duration: "10m", target: 100 }, 
    { duration: "5m", target: 0 }, 
  ],
  thresholds: {
    'http_req_duration': ['p(99)<1500'], 
    'logged in successfully': ['p(99)<1500'], 
  }
};
```

ตอนทำ Load Tests เราควรกำหนด ramp-up stage เสมอ เพื่อเปิดโอกาสให้ระบบของเราวอร์มอัพ หรือ auto scale ได้ทัน อีกทั้งยังให้เราได้เห็นภาพตอนที่โหลดยังไม่เยอะไต่ขึ้นไปตอนที่โหลดเยอะแล้วอีกด้วย

ซึ่งการ ramp-up นั้น เราควรเริ่มจากความเข้มข้นน้อยๆ แล้วค่อยๆ เพิ่มปริมาณความเข้มข้นของการทดสอบไปทีละนิดๆ เพื่อป้องกันไม่ให้ระบบของเราล่มไปทันทีซึ่งนั้นจะเปลี่ยนจาก load test จะกลายเป็น stress testing ไปซะได้

## Stress testing
![image](https://github.com/user-attachments/assets/c0affa0a-07fb-4e64-9cae-9e58adfff11a)

ทดสอบเพื่อดูในเรื่อง availability และ stability ตอนที่ได้รับโหลดเยอะๆ เป็นหลัก หา limit ของระบบเรา และหาคำตอบว่าจะเกิดอะไรขึ้นถ้าเราดันขึ้นไปถึง limit ของระบบเราแล้ว อีกทั้งยังใช้ดูด้วยว่าระบบเราจะกลับมาปกติได้หรือไม่หลังจากจบการทดสอบไปแล้ว

นั่นก็แปลว่าเมื่อเราทำ stress test เราก็จะกำหนดให้เป็นค่าที่มันเกินกว่าค่าที่มันควรจะเป็นค่าปกติ แต่นั่นก็ไม่ได้หมายถึงว่าเราจะโผล่มาแล้วใช้ค่าสูงๆ เลยทีเดียวไม่อย่างนั้นเราจะเรียนว่า spike test แต่เรายังต้องค่อยๆ ramp-up ขึ้นไปเรื่อยๆ เช่นเดียวกับ load test อยู่ เพียงแต่เราจะดันให้มันเกิดนจุดปกติที่ระบบจะรับได้

ตัวอย่างคลาสสิคที่เอาไว้ทดสอบ stress test ก็พวก event ของวันสำคัญๆ ต่างๆ ที่คนจะเข้ามาใช้งานเว็บเราอย่างล้นหลาม เป็นต้น

ยกตัวอย่างเช่น ระบบเรารองรับโหลดได้ปกติที่ 200 VUs และจะถึง limit แถวๆ 400 VUs เราก็ config ได้ดังนี้

```
export let options = {
  stages: [
    { duration: '2m', target: 100 }, // below normal load
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 }, // normal load
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 }, // around the breaking point
    { duration: '5m', target: 300 },
    { duration: '2m', target: 400 }, // beyond the breaking point
    { duration: '5m', target: 400 },
    { duration: '10m', target: 0 }, // scale down. Recovery stage.
  ],
};
```

## Spike Testing
![image](https://github.com/user-attachments/assets/a3a49400-225b-4379-94b3-1afe3d6275ba)

ทดสอบในกรณีที่มีคนเข้าเว็บเราเยอะๆ ในระยะเวลาสั้นๆ เช่น มีโฆษณาสาธาณะแบบ realtime ที่เมื่อคนเห็นแล้วจะต้องรีบเปิดเว็บเราเข้ามาทันที คนก็จะแห่เข้าเว็บเราจำนวนมากๆ พร้อมๆ กัน หรือมีคนดังใน social network ที่มี follower หลักแสนคน เอา link โปรโมชั่นขายของของเว็บเราไปแชร์ ทำให้ follower กดเข้า link นี้พร้อมๆ กัน ซึ่งการ config ให้สอดคล้องกับสถานการณ์ที่ยกมา สามารถทำได้ดังนี้

```
export let options = {
  stages: [
    { duration: '10s', target: 100 }, // below normal load
    { duration: '1m', target: 100 },
    { duration: '10s', target: 1400 }, // spike to 1400 users
    { duration: '3m', target: 1400 }, // stay at 1400 for 3 minutes
    { duration: '10s', target: 100 }, // scale down. Recovery stage.
    { duration: '3m', target: 100 },
    { duration: '10s', target: 0 },
  ],
};
```
