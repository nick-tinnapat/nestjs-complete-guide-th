# คู่มือการเรียนรู้ NestJS ฉบับสมบูรณ์ - ส่วนที่ 1

## สารบัญ
1. [แนะนำ NestJS และการติดตั้ง](#1-แนะนำ-nestjs-และการติดตั้ง)
2. [โครงสร้างโปรเจค](#2-โครงสร้างโปรเจค)
3. [Modules](#3-modules)
4. [Controllers](#4-controllers)
5. [Services](#5-services)
6. [Providers](#6-providers)
7. [Pipes](#7-pipes)

## 1. แนะนำ NestJS และการติดตั้ง

### NestJS คืออะไร?
NestJS เป็น Node.js framework สำหรับสร้าง server-side applications ที่มีประสิทธิภาพและสเกลได้ดี โดยใช้ TypeScript เป็นภาษาหลัก มีการออกแบบโครงสร้างที่ได้รับแรงบันดาลใจจาก Angular ทำให้มีความเป็นระเบียบ และง่ายต่อการบำรุงรักษา

### ข้อดีของ NestJS
- ใช้ TypeScript เป็นหลัก ทำให้มี type safety
- มีโครงสร้างที่ชัดเจน ทำให้โค้ดเป็นระเบียบ
- รองรับ Dependency Injection ทำให้เขียนโค้ดที่ทดสอบได้ง่าย
- มี ecosystem ที่สมบูรณ์ รองรับการทำงานกับเทคโนโลยีอื่นๆ เช่น GraphQL, WebSockets, Microservices

### การติดตั้ง NestJS CLI และสร้างโปรเจคใหม่

ขั้นแรก เราจะติดตั้ง NestJS CLI ซึ่งเป็นเครื่องมือที่ช่วยในการสร้างและจัดการโปรเจค NestJS

```bash
npm i -g @nestjs/cli
```

จากนั้นสร้างโปรเจคใหม่:

```bash
nest new e-learning-api
```

เมื่อรันคำสั่งนี้ CLI จะถามว่าเราต้องการใช้ package manager อะไร (npm, yarn, pnpm) เลือกตามที่คุณถนัด

เมื่อสร้างโปรเจคเสร็จแล้ว เราจะได้โครงสร้างไฟล์พื้นฐานดังนี้:

```
e-learning-api/
├── src/
│   ├── app.controller.spec.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   └── main.ts
├── test/
├── node_modules/
├── package.json
├── tsconfig.json
└── ...
```

ทดลองรันโปรเจค:

```bash
cd e-learning-api
npm run start:dev
```

เมื่อรันแล้ว สามารถเข้าไปที่ `http://localhost:3000` จะเห็นข้อความ "Hello World!" แสดงว่าเราสร้างโปรเจคสำเร็จแล้ว

## 2. โครงสร้างโปรเจค

โครงสร้างโปรเจคของ NestJS มีความเป็นระเบียบและเป็นโมดูลาร์ ทำให้ง่ายต่อการจัดการโค้ด

### ไฟล์สำคัญในโปรเจค
- **main.ts**: จุดเริ่มต้นของแอปพลิเคชัน ที่สร้าง NestJS application และกำหนดค่าต่างๆ
- **app.module.ts**: Root module ของแอปพลิเคชัน
- **app.controller.ts**: Controller พื้นฐาน
- **app.service.ts**: Service พื้นฐาน

### การจัดโครงสร้างโปรเจคตามหลัก Domain-Driven Design

เราจะจัดโครงสร้างโปรเจคตามหลัก Domain-Driven Design โดยแบ่งตาม feature หรือ domain ดังนี้:

```
src/
├── main.ts
├── app.module.ts
├── common/                 # โค้ดที่ใช้ร่วมกัน
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   ├── middleware/
│   └── pipes/
├── config/                 # การตั้งค่าต่างๆ
├── users/                  # โมดูลผู้ใช้งาน
│   ├── dto/
│   ├── entities/
│   ├── users.controller.ts
│   ├── users.service.ts
│   └── users.module.ts
├── courses/                # โมดูลหลักสูตร
│   ├── dto/
│   ├── entities/
│   ├── courses.controller.ts
│   ├── courses.service.ts
│   └── courses.module.ts
├── lessons/                # โมดูลบทเรียน
│   ├── ...
└── enrollments/            # โมดูลการลงทะเบียน
    ├── ...
```

## 3. Modules

Module เป็นส่วนสำคัญของ NestJS ที่ช่วยในการจัดกลุ่มของ controllers, services และ components อื่นๆ ที่เกี่ยวข้องกัน

### การสร้าง Module

เราจะสร้าง module สำหรับผู้ใช้งานก่อน:

```bash
nest generate module users
```

หรือเขียนสั้นๆ:

```bash
nest g mo users
```

คำสั่งนี้จะสร้างไฟล์ `users.module.ts` ในโฟลเดอร์ `users`:

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';

@Module({
  imports: [],
  controllers: [],
  providers: [],
  exports: []
})
export class UsersModule {}
```

### องค์ประกอบของ Module
- **imports**: รายการ modules อื่นๆ ที่ module นี้ต้องการใช้
- **controllers**: รายการ controllers ที่เป็นของ module นี้
- **providers**: รายการ services และ providers อื่นๆ ที่เป็นของ module นี้
- **exports**: รายการ providers ที่ต้องการให้ modules อื่นๆ สามารถเข้าถึงได้

### การสร้าง Module อื่นๆ

ทำเช่นเดียวกันสำหรับ modules อื่นๆ:

```bash
nest g mo courses
nest g mo lessons
nest g mo enrollments
```

### การเชื่อมต่อ Modules เข้ากับ Root Module

เราต้องเพิ่ม modules ที่สร้างขึ้นเข้าไปใน `app.module.ts`:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { UsersModule } from './users/users.module';
import { CoursesModule } from './courses/courses.module';
import { LessonsModule } from './lessons/lessons.module';
import { EnrollmentsModule } from './enrollments/enrollments.module';

@Module({
  imports: [
    UsersModule,
    CoursesModule,
    LessonsModule,
    EnrollmentsModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## 4. Controllers

Controller รับผิดชอบในการจัดการ HTTP requests และส่ง responses กลับไปยังผู้ใช้

### การสร้าง Controller

สร้าง controller สำหรับผู้ใช้งาน:

```bash
nest g controller users
```

คำสั่งนี้จะสร้างไฟล์ `users.controller.ts` และอัพเดท `users.module.ts` โดยอัตโนมัติ:

```typescript
// src/users/users.controller.ts
import { Controller } from '@nestjs/common';

@Controller('users')
export class UsersController {}
```

### การเพิ่ม Routes และ Methods

เพิ่ม routes และ methods ใน controller:

```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Body, Param, Put, Delete } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

### การใช้ Decorators ใน Controller

NestJS มี decorators มากมายที่ช่วยในการจัดการ requests:

- **@Controller()**: กำหนด prefix ของ route
- **@Get(), @Post(), @Put(), @Delete(), @Patch()**: กำหนด HTTP method
- **@Param()**: เข้าถึงพารามิเตอร์ใน URL
- **@Body()**: เข้าถึงข้อมูลใน request body
- **@Query()**: เข้าถึง query parameters
- **@Headers()**: เข้าถึง HTTP headers
- **@Req(), @Res()**: เข้าถึง Express Request และ Response objects

### การสร้าง DTO (Data Transfer Objects)

DTO ใช้สำหรับกำหนดโครงสร้างข้อมูลที่รับเข้ามาและส่งออกไป:

```typescript
// src/users/dto/create-user.dto.ts
export class CreateUserDto {
  readonly email: string;
  readonly password: string;
  readonly firstName: string;
  readonly lastName: string;
}

// src/users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

## 5. Services

Service รับผิดชอบในการจัดการ business logic ของแอปพลิเคชัน

### การสร้าง Service

สร้าง service สำหรับผู้ใช้งาน:

```bash
nest g service users
```

คำสั่งนี้จะสร้างไฟล์ `users.service.ts` และอัพเดท `users.module.ts` โดยอัตโนมัติ:

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  private users = [];

  create(createUserDto: CreateUserDto) {
    const newUser = {
      id: this.users.length + 1,
      ...createUserDto,
    };
    this.users.push(newUser);
    return newUser;
  }

  findAll() {
    return this.users;
  }

  findOne(id: number) {
    return this.users.find(user => user.id === id);
  }

  update(id: number, updateUserDto: UpdateUserDto) {
    const userIndex = this.users.findIndex(user => user.id === id);
    if (userIndex === -1) {
      return null;
    }
    
    const updatedUser = {
      ...this.users[userIndex],
      ...updateUserDto,
    };
    
    this.users[userIndex] = updatedUser;
    return updatedUser;
  }

  remove(id: number) {
    const userIndex = this.users.findIndex(user => user.id === id);
    if (userIndex === -1) {
      return null;
    }
    
    const removedUser = this.users[userIndex];
    this.users.splice(userIndex, 1);
    return removedUser;
  }
}
```

### การใช้ Dependency Injection

NestJS ใช้ Dependency Injection ในการจัดการการสร้างและการใช้งาน services และ providers อื่นๆ

ตัวอย่างการใช้ service ใน controller:

```typescript
// src/users/users.controller.ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}
  
  // ...methods
}
```

## 6. Providers

Provider เป็นแนวคิดพื้นฐานใน NestJS ซึ่ง service ก็เป็น provider ประเภทหนึ่ง นอกจากนี้ยังมี providers อื่นๆ เช่น repositories, factories, helpers เป็นต้น

### การสร้าง Custom Provider

สร้าง provider สำหรับการส่งอีเมล:

```typescript
// src/common/providers/email.provider.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class EmailService {
  async sendWelcomeEmail(email: string, name: string) {
    console.log(`Sending welcome email to ${name} at ${email}`);
    // ในโปรเจคจริง จะใช้ library เช่น nodemailer ในการส่งอีเมล
    return true;
  }
}
```

### การใช้ Custom Provider

เพิ่ม provider ใน module:

```typescript
// src/common/common.module.ts
import { Module } from '@nestjs/common';
import { EmailService } from './providers/email.provider';

@Module({
  providers: [EmailService],
  exports: [EmailService], // ส่งออกเพื่อให้ modules อื่นๆ สามารถใช้งานได้
})
export class CommonModule {}
```

ใช้ provider ใน service:

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { EmailService } from '../common/providers/email.provider';

@Injectable()
export class UsersService {
  constructor(private readonly emailService: EmailService) {}

  async create(createUserDto: CreateUserDto) {
    const newUser = {
      id: this.users.length + 1,
      ...createUserDto,
    };
    this.users.push(newUser);
    
    // ส่งอีเมลต้อนรับ
    await this.emailService.sendWelcomeEmail(
      newUser.email,
      `${newUser.firstName} ${newUser.lastName}`
    );
    
    return newUser;
  }
  
  // ...other methods
}
```

อย่าลืมเพิ่ม CommonModule ใน imports ของ UsersModule:

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { CommonModule } from '../common/common.module';

@Module({
  imports: [CommonModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## 7. Pipes

Pipe ใช้สำหรับการแปลงหรือตรวจสอบข้อมูลที่รับเข้ามาก่อนที่จะส่งไปยัง handler

### Built-in Pipes

NestJS มี built-in pipes หลายตัว เช่น:
- **ValidationPipe**: ใช้สำหรับตรวจสอบข้อมูลตาม DTO
- **ParseIntPipe**: แปลงข้อมูลเป็น integer
- **ParseBoolPipe**: แปลงข้อมูลเป็น boolean
- **ParseArrayPipe**: แปลงข้อมูลเป็น array
- **ParseUUIDPipe**: ตรวจสอบว่าข้อมูลเป็น UUID ที่ถูกต้อง

ตัวอย่างการใช้ ParseIntPipe:

```typescript
// src/users/users.controller.ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.usersService.findOne(id);
}
```

### การใช้ ValidationPipe

เพิ่ม ValidationPipe ใน main.ts เพื่อใช้งานทั่วทั้งแอปพลิเคชัน:

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true, // ลบ properties ที่ไม่ได้ระบุใน DTO
    forbidNonWhitelisted: true, // ส่ง error ถ้ามี properties ที่ไม่ได้ระบุใน DTO
    transform: true, // แปลงข้อมูลให้ตรงกับ type ใน DTO
  }));
  await app.listen(3000);
}
bootstrap();
```

### การใช้ class-validator และ class-transformer

ติดตั้ง packages:

```bash
npm install class-validator class-transformer
```

ใช้ decorators จาก class-validator ใน DTO:

```typescript
// src/users/dto/create-user.dto.ts
import { IsEmail, IsNotEmpty, IsString, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  @IsNotEmpty()
  readonly email: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(8)
  readonly password: string;

  @IsString()
  @IsNotEmpty()
  readonly firstName: string;

  @IsString()
  @IsNotEmpty()
  readonly lastName: string;
}
```

### การสร้าง Custom Pipe

สร้าง custom pipe สำหรับตรวจสอบว่าผู้ใช้งานมีอยู่ในระบบหรือไม่:

```typescript
// src/users/pipes/user-exists.pipe.ts
import { PipeTransform, Injectable, NotFoundException } from '@nestjs/common';
import { UsersService } from '../users.service';

@Injectable()
export class UserExistsPipe implements PipeTransform {
  constructor(private readonly usersService: UsersService) {}

  async transform(value: number) {
    const user = await this.usersService.findOne(value);
    if (!user) {
      throw new NotFoundException(`User with ID ${value} not found`);
    }
    return value;
  }
}
```

ใช้ custom pipe ใน controller:

```typescript
// src/users/users.controller.ts
@Get(':id')
findOne(
  @Param('id', ParseIntPipe, UserExistsPipe) id: number
) {
  return this.usersService.findOne(id);
}
```
