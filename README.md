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

![image](https://github.com/thangdtph27626/spring-Batch/assets/109157942/6e13e826-3f0e-4385-b3e2-0a1df0d4fd0a)

- Cụ thể là ở đây chúng ta sẽ xử lý avg mark(điểm trung bình) để từ đó tìm ra xếp loại của sinh viên đó và lưu chúng vào trong database. Nào, bây giờ chúng ta hãy giải quyết bài toán này nào :D (với IDE eclipse).

### 3.2 . Tổng quát Project 

![image](https://github.com/thangdtph27626/spring-Batch/assets/109157942/6c60b359-a5da-47a6-ae91-29ed2259d289)

### 3.3 . Maven Dependencies

Trong file pom, chúng ta sẽ dùng các dependencies sau:

```
<dependencies>
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-batch</artifactId>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
		</dependency>
</dependencies>
```
### 3.4 . Tạo resources

Trong thư mục src/main/resources, ta tạo các files sau đây:

- information-list.csv :

```
Nam ,SE1401 ,7.3
Tin ,SE1402 ,8.2
Khang ,SE1403 ,4.6
Hien ,SE1401 ,9.3
Huy ,SE1409 ,6.9
```

- application.properties :

```
file.input=information-list.csv

```
- schema-all.sql :

```
drop table information if exists;

create table information (
	student_id bigint identity primary key, 
	name varchar(20) not null,
	class varchar(20),
	avg_mark double not null, 
	classification varchar(30) not null
)
```

### 3.5 . Tạo Model

- Trong package model, ta tạo class Information dùng để lưu và xử lý các thông tin của sinh viên. 
```
@AllArgsConstructor
@NoArgsConstructor
@Data
public class Information {
	private String name;
	private String classes;
	private double avgMark;
	private String classification;
}
```

### 3.5 . Job Configuration
```
@Configuration
@EnableBatchProcessing
public class BatchConfiguration {
	@Autowired
	public JobBuilderFactory jobBuilderFactory;

	@Autowired
	public StepBuilderFactory stepBuilderFactory;

	@Value("${file.input}")
	private String fileInput;
        ....
}
```

 Trong package config , ta sẽ tạo class BatchConfiguration và add 2 anotitation @Configuration và @EnableBatchProcessing để phục vụ cho việc config cho project. 

Cụ thể là @EnableBatchProcessing cung cấp base configuration để building batch job. Khi đó, các beans sau sẽ được tạo ra để chúng ta có thể autowired bất cứ lúc nào :

- JobRepository: bean name "jobRepository"

- JobLauncher: bean name "jobLauncher"

- JobRegistry: bean name "jobRegistry"

- PlatformTransactionManager: bean name "transactionManager"

- JobBuilderFactory: bean name "jobBuilders"

- StepBuilderFactory: bean name "stepBuilders"

 Trong project này,chúng ta sẽ dùng JobBuilderFactory và StepBuilderFactory là 2 factories cực kì hữu ích để build job configuration và jobs steps.

 ### 3.6 . Tạo reader, processor và writer cho Job. 

Tiếp tục ở trong class BatchConfiguration ta sẽ tạo reader và writer cho Job với 2 Bean methods là 

+ reader() : Để đọc information-list.csv  -> Information object. 

+ writer(DataSource dataSource) : Write mỗi information item vào trong database, trong đó dataSource đã được khởi tạo tự động bằng anotation @EnableBatchProcessing theo những config trong file pom và schema-all.sql .

```
  @Bean
	public FlatFileItemReader<Information> reader() {
		return new FlatFileItemReaderBuilder<Information>().name("InformationReader")
				.resource(new ClassPathResource(fileInput)).delimited()
				.names(new String[] { "name", "classes", "avgMark" })
				.fieldSetMapper(new BeanWrapperFieldSetMapper<Information>() {
					{
						setTargetType(Information.class);
					}
				}).build();
	}

	@Bean
	public JdbcBatchItemWriter<Information> writer(DataSource dataSource) {
		return new JdbcBatchItemWriterBuilder<Information>()
				.itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
				.sql("INSERT INTO information (name, class, avg_mark, classification) VALUES (:name, :classes, :avgMark, :classification)")
				.dataSource(dataSource).build();
	}

```

- Để xử lý data sau khi đọc từ file csv, chúng ta sẽ tạo custom processor với class InforItemProcessor trong package processor như sau :
  
 ```
public class InforItemProcessor implements ItemProcessor<Information, Information>{
	private static final Logger LOGGER = LoggerFactory.getLogger(InforItemProcessor.class);

	@Override
	public Information process(final Information information) throws Exception {
		String name = information.getName().toUpperCase();
		String classes = information.getClasses();
		double avgMark = information.getAvgMark(); 
		
		String classification = "Excellent"; 
		if (avgMark<0 || avgMark>10) {
			classification = "Error! Please check input again.";
		}else if (avgMark<6) {
			classification = "Failing"; 
		}else if (avgMark<7) {
			classification = "Bellow Average"; 
		}else if (avgMark<8) {
			classification = "Average"; 
		}else if (avgMark<9) {
			classification = "Very good";
		}
		Information transformedInformation = new Information(name, classes, avgMark, classification); 
		LOGGER.info("Converting ( {} ) into ( {} )", information, transformedInformation);
		
		return transformedInformation;
	}	
}
```

 Sau đó định nghĩa Bean method của processor trong class BatchConfiguration :
 
 ```
@Bean
	public InforItemProcessor processor() {
		return new InforItemProcessor();
	}
```

### 3.7 . Tạo JobCompletionNotificationListener 

- Với mục đích để check xem Job của chúng ta đã tạo ra hoạt động tốt hay chưa, ta sẽ tạo class JobCompletionNotificationListener được extend từ class JobExecutionListenerSupport để select thông tin từ table information như sau :

```
@Component
public class JobCompletionNotificationListener extends JobExecutionListenerSupport {
	private static final Logger LOGGER = LoggerFactory.getLogger(JobCompletionNotificationListener.class);
	private final JdbcTemplate jdbcTemplate;

	@Autowired
	public JobCompletionNotificationListener(JdbcTemplate jdbcTemplate) {
		this.jdbcTemplate = jdbcTemplate;
	}

	@Override
	public void afterJob(JobExecution jobExecution) {
		if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
			LOGGER.info("Job finished! Time to verify the results");

			String query = "SELECT name, class, avg_mark, classification FROM information";
			jdbcTemplate.query(query,
					(rs, row) -> new Information(rs.getString(1), rs.getString(2), rs.getDouble(3), rs.getString(4)))
					.forEach(infor -> LOGGER.info("Found < {} > in database.", infor));
		}
	}
}
```

- @Component là một Annotation (chú thích) đánh dấu trên các Class để giúp Spring biết nó là một Bean.

- Hàm afterJob sẽ tự động được thực thi ngay sau khi Job của chúng ta kết thúc. 

   + JobExecution là "primary storage mechanism" thể hiện cho chúng ta thấy tất cả những gì sẽ diễn ra suốt quá trình chạy Job.

   + Status chính là một property của trong JobExcution, được thể hiện qua BatchStatus - là một enumeration gồm các values sau :

[ COMPLETED, STARTING, STARTED, STOPPING, STOPPED, FAILED, ABANDONED, UNKNOWN ]

Trong trường hợp của project : Nếu status là 'COMPLETED' thì ta sẽ thực hiện query để lấy ra thông tin trong table information.

### 3.8 . Cấu hình Step và Job

- Cuối cùng, vẫn trong class BatchConfiguration chúng ta sẽ tạo config cho Step và Job :

```
 @Bean
	public Job importUserJob(JobCompletionNotificationListener listener, Step step1) {
		return jobBuilderFactory.get("importUserJob")
				.incrementer(new RunIdIncrementer())
				.listener(listener)
				.flow(step1)
				.end()
				.build();
	}
	
	@Bean
	public Step step1(JdbcBatchItemWriter<Information> writer) {
		return stepBuilderFactory.get("step1")
	            .<Information, Information> chunk(10)
	            .reader(reader())
	            .processor(processor())
	            .writer(writer)
	            .build();
	}
```

- Step :

   + Được đặt tên là 'step1' 

   + chunk(10) ở đây nghĩa là cứ [read + process] liên tục 10 record thì sẽ write kết quả vào trong database.

   + Với reader, processor và writer mà ta đã tạo ra ở trước đó.

Các bạn nên lưu ý là trong 1 step chỉ có duy nhất một reader, processor và writer nhưng một Job lại có thể chứa nhiều Step. 

- Job :

   + Đặt tên là 'importUserJob' 

   + incrementer(new RunIdIncrementer()) nghĩa là JobParameter sẽ được định nghĩa là (run.id)++ với khởi tạo là : {run.id=1} ,

   + Đăng ký một listener với một custom listener chính là class JobCompletionNotificationListener đã tạo ra ở phần 3.7.

   + Thực thi step1 .
 
### 3.4.4 . Chạy Job

```
2021-02-28 15:34:18.736  INFO 16364 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [FlowJob: [name=importUserJob]] launched with the following parameters: [{run.id=1}]
2021-02-28 15:34:18.792  INFO 16364 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step1]
2021-02-28 15:34:18.840  INFO 16364 --- [           main] s.processor.InforItemProcessor           : Converting ( Information(name=Nam, classes=SE1401, avgMark=7.3, classification=null) into ( Information(name=NAM, classes=SE1401, avgMark=7.3, classification=Average) )
2021-02-28 15:34:18.841  INFO 16364 --- [           main] s.processor.InforItemProcessor           : Converting ( Information(name=Tin, classes=SE1402, avgMark=8.2, classification=null) into ( Information(name=TIN, classes=SE1402, avgMark=8.2, classification=Very good) )
2021-02-28 15:34:18.841  INFO 16364 --- [           main] s.processor.InforItemProcessor           : Converting ( Information(name=Khang, classes=SE1403, avgMark=4.6, classification=null) into ( Information(name=KHANG, classes=SE1403, avgMark=4.6, classification=Failing) )
2021-02-28 15:34:18.841  INFO 16364 --- [           main] s.processor.InforItemProcessor           : Converting ( Information(name=Hien, classes=SE1401, avgMark=9.3, classification=null) into ( Information(name=HIEN, classes=SE1401, avgMark=9.3, classification=Excellent) )
2021-02-28 15:34:18.841  INFO 16364 --- [           main] s.processor.InforItemProcessor           : Converting ( Information(name=Huy, classes=SE1409, avgMark=6.9, classification=null) into ( Information(name=HUY, classes=SE1409, avgMark=6.9, classification=Bellow Average) )
2021-02-28 15:34:18.854  INFO 16364 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step1] executed in 62ms
2021-02-28 15:34:18.860  INFO 16364 --- [           main] s.c.JobCompletionNotificationListener    : Job finished! Time to verify the results
2021-02-28 15:34:18.863  INFO 16364 --- [           main] s.c.JobCompletionNotificationListener    : Found < Information(name=NAM, classes=SE1401, avgMark=7.3, classification=Average) > in database.
2021-02-28 15:34:18.863  INFO 16364 --- [           main] s.c.JobCompletionNotificationListener    : Found < Information(name=TIN, classes=SE1402, avgMark=8.2, classification=Very good) > in database.
2021-02-28 15:34:18.863  INFO 16364 --- [           main] s.c.JobCompletionNotificationListener    : Found < Information(name=KHANG, classes=SE1403, avgMark=4.6, classification=Failing) > in database.
2021-02-28 15:34:18.863  INFO 16364 --- [           main] s.c.JobCompletionNotificationListener    : Found < Information(name=HIEN, classes=SE1401, avgMark=9.3, classification=Excellent) > in database.
2021-02-28 15:34:18.864  INFO 16364 --- [           main] s.c.JobCompletionNotificationListener    : Found < Information(name=HUY, classes=SE1409, avgMark=6.9, classification=Bellow Average) > in database.
2021-02-28 15:34:18.866  INFO 16364 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [FlowJob: [name=importUserJob]] completed with the following parameters: [{run.id=1}] and the following status: [COMPLETED] in 81ms
```

