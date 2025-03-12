# คู่มือการเรียนรู้ NestJS ฉบับสมบูรณ์ - ส่วนที่ 3

## สารบัญ
1. [การทำ Relationships ใน TypeORM](#15-การทำ-relationships-ใน-typeorm)
2. [การทำ Transactions](#16-การทำ-transactions)
3. [การทำ Pagination](#17-การทำ-pagination)
4. [การทำ Caching](#18-การทำ-caching)
5. [การทำ File Upload](#19-การทำ-file-upload)
6. [การทำ Logging](#20-การทำ-logging)
7. [การทำ Testing](#21-การทำ-testing)

## 15. การทำ Relationships ใน TypeORM

TypeORM รองรับความสัมพันธ์ (relationships) หลายประเภท เช่น One-to-One, One-to-Many, Many-to-One, และ Many-to-Many

### One-to-One Relationship

สร้าง entity สำหรับ Profile:

```typescript
// src/users/entities/profile.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, OneToOne, JoinColumn } from 'typeorm';
import { User } from './user.entity';

@Entity()
export class Profile {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ nullable: true })
  bio: string;

  @Column({ nullable: true })
  website: string;

  @Column({ nullable: true })
  phoneNumber: string;

  @Column({ nullable: true })
  address: string;

  @OneToOne(() => User, user => user.profile)
  @JoinColumn()
  user: User;
}
```

อัพเดท User entity:

```typescript
// src/users/entities/user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, OneToOne } from 'typeorm';
import { Profile } from './profile.entity';

@Entity()
export class User {
  // ...existing fields

  @OneToOne(() => Profile, profile => profile.user, { cascade: true })
  profile: Profile;
}
```

### One-to-Many และ Many-to-One Relationship

สร้าง entity สำหรับ Course:

```typescript
// src/courses/entities/course.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, ManyToOne, OneToMany } from 'typeorm';
import { User } from '../../users/entities/user.entity';
import { Lesson } from '../../lessons/entities/lesson.entity';

@Entity()
export class Course {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column()
  description: string;

  @Column({ default: 0 })
  price: number;

  @Column({ default: true })
  isPublished: boolean;

  @ManyToOne(() => User, user => user.courses)
  instructor: User;

  @OneToMany(() => Lesson, lesson => lesson.course, { cascade: true })
  lessons: Lesson[];
}
```

สร้าง entity สำหรับ Lesson:

```typescript
// src/lessons/entities/lesson.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, ManyToOne } from 'typeorm';
import { Course } from '../../courses/entities/course.entity';

@Entity()
export class Lesson {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column()
  content: string;

  @Column({ default: 0 })
  duration: number;

  @ManyToOne(() => Course, course => course.lessons)
  course: Course;
}
```

อัพเดท User entity:

```typescript
// src/users/entities/user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, OneToOne, OneToMany } from 'typeorm';
import { Profile } from './profile.entity';
import { Course } from '../../courses/entities/course.entity';

@Entity()
export class User {
  // ...existing fields

  @OneToOne(() => Profile, profile => profile.user, { cascade: true })
  profile: Profile;

  @OneToMany(() => Course, course => course.instructor)
  courses: Course[];
}
```

### Many-to-Many Relationship

สร้าง entity สำหรับ Enrollment:

```typescript
// src/enrollments/entities/enrollment.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, ManyToOne, CreateDateColumn } from 'typeorm';
import { User } from '../../users/entities/user.entity';
import { Course } from '../../courses/entities/course.entity';

@Entity()
export class Enrollment {
  @PrimaryGeneratedColumn()
  id: number;

  @ManyToOne(() => User, user => user.enrollments)
  student: User;

  @ManyToOne(() => Course)
  course: Course;

  @Column({ default: false })
  isCompleted: boolean;

  @CreateDateColumn()
  enrolledAt: Date;
}
```

อัพเดท User entity:

```typescript
// src/users/entities/user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, OneToOne, OneToMany } from 'typeorm';
import { Profile } from './profile.entity';
import { Course } from '../../courses/entities/course.entity';
import { Enrollment } from '../../enrollments/entities/enrollment.entity';

@Entity()
export class User {
  // ...existing fields

  @OneToOne(() => Profile, profile => profile.user, { cascade: true })
  profile: Profile;

  @OneToMany(() => Course, course => course.instructor)
  courses: Course[];

  @OneToMany(() => Enrollment, enrollment => enrollment.student)
  enrollments: Enrollment[];
}
```

### การใช้ Relations ใน Query

การใช้ relations ใน query:

```typescript
// src/courses/courses.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Course } from './entities/course.entity';

@Injectable()
export class CoursesService {
  constructor(
    @InjectRepository(Course)
    private coursesRepository: Repository<Course>,
  ) {}

  async findOne(id: number): Promise<Course> {
    return this.coursesRepository.findOne({
      where: { id },
      relations: ['instructor', 'lessons'],
    });
  }

  async findAllWithInstructor(): Promise<Course[]> {
    return this.coursesRepository.find({
      relations: ['instructor'],
    });
  }
}
```

## 16. การทำ Transactions

Transaction ใช้สำหรับการทำงานกับฐานข้อมูลที่ต้องการความสอดคล้องกัน (consistency) เช่น การทำงานหลายอย่างที่ต้องสำเร็จทั้งหมดหรือล้มเหลวทั้งหมด

### การใช้ Transaction ใน Service

```typescript
// src/enrollments/enrollments.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, DataSource } from 'typeorm';
import { Enrollment } from './entities/enrollment.entity';
import { User } from '../users/entities/user.entity';
import { Course } from '../courses/entities/course.entity';

@Injectable()
export class EnrollmentsService {
  constructor(
    @InjectRepository(Enrollment)
    private enrollmentsRepository: Repository<Enrollment>,
    @InjectRepository(User)
    private usersRepository: Repository<User>,
    @InjectRepository(Course)
    private coursesRepository: Repository<Course>,
    private dataSource: DataSource,
  ) {}

  async enrollStudentToCourse(studentId: number, courseId: number): Promise<Enrollment> {
    // เริ่ม transaction
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // ตรวจสอบว่ามีผู้ใช้งานและหลักสูตรที่ต้องการหรือไม่
      const student = await this.usersRepository.findOne({ where: { id: studentId } });
      const course = await this.coursesRepository.findOne({ where: { id: courseId } });

      if (!student || !course) {
        throw new Error('Student or course not found');
      }

      // สร้าง enrollment
      const enrollment = new Enrollment();
      enrollment.student = student;
      enrollment.course = course;

      // บันทึก enrollment
      const savedEnrollment = await queryRunner.manager.save(enrollment);

      // ถ้าทุกอย่างสำเร็จ ให้ commit transaction
      await queryRunner.commitTransaction();

      return savedEnrollment;
    } catch (error) {
      // ถ้าเกิดข้อผิดพลาด ให้ rollback transaction
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      // ปล่อย queryRunner
      await queryRunner.release();
    }
  }
}
```

## 17. การทำ Pagination

การทำ pagination ช่วยให้สามารถแบ่งข้อมูลออกเป็นหน้าๆ ได้

### การสร้าง Pagination DTO

```typescript
// src/common/dto/pagination.dto.ts
import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  limit?: number = 10;
}
```

### การใช้ Pagination ใน Service

```typescript
// src/courses/courses.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Course } from './entities/course.entity';
import { PaginationDto } from '../common/dto/pagination.dto';

@Injectable()
export class CoursesService {
  constructor(
    @InjectRepository(Course)
    private coursesRepository: Repository<Course>,
  ) {}

  async findAll(paginationDto: PaginationDto): Promise<{ data: Course[]; total: number; page: number; limit: number }> {
    const { page, limit } = paginationDto;
    const skip = (page - 1) * limit;

    const [data, total] = await this.coursesRepository.findAndCount({
      skip,
      take: limit,
      order: { id: 'DESC' },
    });

    return {
      data,
      total,
      page,
      limit,
    };
  }
}
```

### การใช้ Pagination ใน Controller

```typescript
// src/courses/courses.controller.ts
import { Controller, Get, Query } from '@nestjs/common';
import { CoursesService } from './courses.service';
import { PaginationDto } from '../common/dto/pagination.dto';

@Controller('courses')
export class CoursesController {
  constructor(private readonly coursesService: CoursesService) {}

  @Get()
  findAll(@Query() paginationDto: PaginationDto) {
    return this.coursesService.findAll(paginationDto);
  }
}
```

## 18. การทำ Caching

การทำ caching ช่วยเพิ่มประสิทธิภาพของแอปพลิเคชันโดยการจัดเก็บข้อมูลที่ใช้บ่อยไว้ในหน่วยความจำ

### การติดตั้ง Cache Manager

```bash
npm install @nestjs/cache-manager cache-manager
```

### การตั้งค่า Cache Module

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { AppService } from './app.service';
// ...other imports

@Module({
  imports: [
    CacheModule.register({
      ttl: 60000, // time to live in milliseconds
      max: 100, // maximum number of items in cache
    }),
    // ...other modules
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### การใช้ Cache ใน Service

```typescript
// src/courses/courses.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Course } from './entities/course.entity';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CoursesService {
  constructor(
    @InjectRepository(Course)
    private coursesRepository: Repository<Course>,
    @Inject(CACHE_MANAGER)
    private cacheManager: Cache,
  ) {}

  async findAll(): Promise<Course[]> {
    // ตรวจสอบว่ามีข้อมูลใน cache หรือไม่
    const cachedCourses = await this.cacheManager.get<Course[]>('all_courses');
    if (cachedCourses) {
      return cachedCourses;
    }

    // ถ้าไม่มีข้อมูลใน cache ให้ดึงข้อมูลจากฐานข้อมูล
    const courses = await this.coursesRepository.find();
    
    // เก็บข้อมูลลง cache
    await this.cacheManager.set('all_courses', courses);
    
    return courses;
  }

  async findOne(id: number): Promise<Course> {
    // ตรวจสอบว่ามีข้อมูลใน cache หรือไม่
    const cachedCourse = await this.cacheManager.get<Course>(`course_${id}`);
    if (cachedCourse) {
      return cachedCourse;
    }

    // ถ้าไม่มีข้อมูลใน cache ให้ดึงข้อมูลจากฐานข้อมูล
    const course = await this.coursesRepository.findOne({ where: { id } });
    
    // เก็บข้อมูลลง cache
    await this.cacheManager.set(`course_${id}`, course);
    
    return course;
  }

  async create(createCourseDto: any): Promise<Course> {
    const course = this.coursesRepository.create(createCourseDto);
    const savedCourse = await this.coursesRepository.save(course);
    
    // ล้าง cache เมื่อมีการสร้างข้อมูลใหม่
    await this.cacheManager.del('all_courses');
    
    return savedCourse;
  }

  async update(id: number, updateCourseDto: any): Promise<Course> {
    await this.coursesRepository.update(id, updateCourseDto);
    const updatedCourse = await this.coursesRepository.findOne({ where: { id } });
    
    // ล้าง cache เมื่อมีการอัพเดทข้อมูล
    await this.cacheManager.del(`course_${id}`);
    await this.cacheManager.del('all_courses');
    
    return updatedCourse;
  }

  async remove(id: number): Promise<void> {
    await this.coursesRepository.delete(id);
    
    // ล้าง cache เมื่อมีการลบข้อมูล
    await this.cacheManager.del(`course_${id}`);
    await this.cacheManager.del('all_courses');
  }
}
```

## 19. การทำ File Upload

### การติดตั้ง Multer

```bash
npm install -D @types/multer
```

### การสร้าง File Upload Controller

```typescript
// src/files/files.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, Get, Param, Res } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname } from 'path';
import { Response } from 'express';

@Controller('files')
export class FilesController {
  @Post('upload')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './uploads',
        filename: (req, file, callback) => {
          const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
          const ext = extname(file.originalname);
          const filename = `${uniqueSuffix}${ext}`;
          callback(null, filename);
        },
      }),
    }),
  )
  uploadFile(@UploadedFile() file: Express.Multer.File) {
    return {
      originalname: file.originalname,
      filename: file.filename,
    };
  }

  @Get(':filename')
  getFile(@Param('filename') filename: string, @Res() res: Response) {
    res.sendFile(filename, { root: './uploads' });
  }
}
```

### การใช้ File Upload ใน Module

```typescript
// src/files/files.module.ts
import { Module } from '@nestjs/common';
import { FilesController } from './files.controller';
import { MulterModule } from '@nestjs/platform-express';

@Module({
  imports: [
    MulterModule.register({
      dest: './uploads',
    }),
  ],
  controllers: [FilesController],
})
export class FilesModule {}
```

## 20. การทำ Logging

### การติดตั้ง Winston

```bash
npm install nest-winston winston
```

### การตั้งค่า Logger

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: WinstonModule.createLogger({
      transports: [
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.ms(),
            winston.format.colorize(),
            winston.format.printf(
              (info) => `${info.timestamp} ${info.level}: ${info.message}`,
            ),
          ),
        }),
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.json(),
          ),
        }),
        new winston.transports.File({
          filename: 'logs/combined.log',
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.json(),
          ),
        }),
      ],
    }),
  });
  await app.listen(3000);
}
bootstrap();
```

### การใช้ Logger ใน Service

```typescript
// src/users/users.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async findAll(): Promise<User[]> {
    this.logger.log('Finding all users');
    return this.usersRepository.find();
  }

  async findOne(id: number): Promise<User> {
    this.logger.log(`Finding user with id ${id}`);
    const user = await this.usersRepository.findOne({ where: { id } });
    if (!user) {
      this.logger.error(`User with id ${id} not found`);
      throw new Error(`User with id ${id} not found`);
    }
    return user;
  }

  // ...other methods
}
```

## 21. การทำ Testing

### Unit Testing

```typescript
// src/users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './entities/user.entity';

describe('UsersService', () => {
  let service: UsersService;
  let mockRepository;

  beforeEach(async () => {
    mockRepository = {
      find: jest.fn(),
      findOne: jest.fn(),
      create: jest.fn(),
      save: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findAll', () => {
    it('should return an array of users', async () => {
      const users = [{ id: 1, email: 'test@example.com' }];
      mockRepository.find.mockResolvedValue(users);

      expect(await service.findAll()).toEqual(users);
      expect(mockRepository.find).toHaveBeenCalled();
    });
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      const user = { id: 1, email: 'test@example.com' };
      mockRepository.findOne.mockResolvedValue(user);

      expect(await service.findOne(1)).toEqual(user);
      expect(mockRepository.findOne).toHaveBeenCalledWith({ where: { id: 1 } });
    });

    it('should throw an error if user not found', async () => {
      mockRepository.findOne.mockResolvedValue(null);

      await expect(service.findOne(1)).rejects.toThrow();
    });
  });
});
```

### E2E Testing

```typescript
// test/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from './../src/app.module';

describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/users (GET)', () => {
    return request(app.getHttpServer())
      .get('/users')
      .expect(200)
      .expect((res) => {
        expect(Array.isArray(res.body)).toBe(true);
      });
  });

  it('/users/:id (GET)', () => {
    return request(app.getHttpServer())
      .get('/users/1')
      .expect(200)
      .expect((res) => {
        expect(res.body).toHaveProperty('id');
        expect(res.body).toHaveProperty('email');
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

## สรุป

ในคู่มือนี้ เราได้เรียนรู้เกี่ยวกับ NestJS ตั้งแต่พื้นฐานไปจนถึงการใช้งานขั้นสูง ครอบคลุมหัวข้อต่างๆ ดังนี้:

1. แนะนำ NestJS และการติดตั้ง
2. โครงสร้างโปรเจค
3. Modules
4. Controllers
5. Services
6. Providers
7. Pipes
8. Guards
9. Interceptors
10. Middleware
11. Exception Filters
12. การเชื่อมต่อฐานข้อมูล
13. การทำ Authentication และ Authorization
14. การทำ Validation
15. การทำ Relationships ใน TypeORM
16. การทำ Transactions
17. การทำ Pagination
18. การทำ Caching
19. การทำ File Upload
20. การทำ Logging
21. การทำ Testing

NestJS เป็น framework ที่มีประสิทธิภาพและยืดหยุ่น เหมาะสำหรับการพัฒนาแอปพลิเคชันขนาดใหญ่ที่ต้องการความเป็นระเบียบและง่ายต่อการบำรุงรักษา

หวังว่าคู่มือนี้จะช่วยให้คุณเข้าใจ NestJS มากขึ้นและสามารถนำไปใช้ในโปรเจคของคุณได้อย่างมีประสิทธิภาพ
