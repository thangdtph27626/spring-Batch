# Giới thiệu về Spring Batch

## 1. Spring Batch là gì?

 - Spring Batch is a lightweight, comprehensive batch framework designed to enable the development of robust batch applications vital for the daily operations of enterprise systems.
 - Spring Batch provides reusable functions that are essential in processing large volumes of records, including logging/tracing, transaction management, job processing statistics, job restart, skip, and resource management.

> Chú ý: Spring Batch bản thân nó không phải là một scheduling framework (như Quartz, Tivoli, Control-M, etc.) nhưng chúng sẽ làm việc cùng nhau.

  ## 2. Thành phần cơ bản. 

![image](https://github.com/thangdtph27626/spring-Batch/assets/109157942/8c37bb40-d7e9-4219-a9b9-2ecb17fc8fe3)

- Diagram trên là một version đơn giản của một kiến trúc batch mà đã được sử dụng qua nhiều thập kỉ. Mình sẽ không nói rõ về từng thành phần nhưng các bạn có thể tham khảo [tại đây](https://docs.spring.io/spring-batch/docs/current/reference/html/index-single.html?fbclid=IwAR2WB9RvKhZw80zsk-0R_hlc9eu9ynJqGHzGSN_s_Z-Q_wwC27-RDBSzKQ8#domainLanguageOfBatch).

- Chỉ cần nhớ 1 số điều sau : 

   + Một Job có thể có một hoặc nhiều Step, nhưng mỗi Step chỉ có duy nhất một ItemReader, một ItemProcessor và một ItemWriter.

   + Khi một Job muốn được launched phải thông qua JobLauncher và data mô tả toàn bộ quá trình thực hiện sẽ được lưu trong JobRepository.

## 3. Ví dụ 


### 3.1 . Yêu cầu: 

- Build một job có thể làm công việc sau:

      Import 1 file .csv chứa thông tin của sinh viên FPT, sau đó sử lý những thông tin đó và lưu kết quả trong in-memory database.

- Sequence diagram để thực thi job :

