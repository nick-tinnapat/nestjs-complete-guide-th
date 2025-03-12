# คู่มือการเรียนรู้ NestJS ฉบับสมบูรณ์ - ส่วนที่ 2

## สารบัญ
1. [Guards](#8-guards)
2. [Interceptors](#9-interceptors)
3. [Middleware](#10-middleware)
4. [Exception Filters](#11-exception-filters)
5. [การเชื่อมต่อฐานข้อมูล](#12-การเชื่อมต่อฐานข้อมูล)
6. [การทำ Authentication และ Authorization](#13-การทำ-authentication-และ-authorization)
7. [การทำ Validation](#14-การทำ-validation)

## 8. Guards

Guard ใช้สำหรับการตรวจสอบว่า request สามารถดำเนินการต่อไปได้หรือไม่ มักใช้สำหรับการตรวจสอบการยืนยันตัวตน (authentication) และการอนุญาต (authorization)

### การสร้าง Authentication Guard

สร้าง guard สำหรับตรวจสอบการยืนยันตัวตน:

```typescript
// src/common/guards/auth.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    
    // ในโปรเจคจริง จะตรวจสอบ JWT token หรือ session
    const authHeader = request.headers.authorization;
    if (!authHeader) {
      throw new UnauthorizedException('Authorization header is missing');
    }
    
    // ตรวจสอบว่า token ถูกต้องหรือไม่
    // ...
    
    return true;
  }
}
```

### การใช้ Guard ใน Controller

ใช้ guard ใน controller:

```typescript
// src/users/users.controller.ts
import { Controller, UseGuards } from '@nestjs/common';
import { AuthGuard } from '../common/guards/auth.guard';

@Controller('users')
@UseGuards(AuthGuard) // ใช้ guard กับทุก route ใน controller
export class UsersController {
  // ...methods
}
```

หรือใช้กับเฉพาะ route:

```typescript
// src/users/users.controller.ts
@Get(':id')
@UseGuards(AuthGuard)
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.usersService.findOne(id);
}
```

### การสร้าง Role-based Guard

สร้าง guard สำหรับตรวจสอบบทบาท (role):

```typescript
// src/common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user; // ต้องมีการเพิ่ม user ใน request ก่อนหน้านี้
    
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

สร้าง decorator สำหรับกำหนด roles:

```typescript
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

ใช้ RolesGuard และ Roles decorator:

```typescript
// src/users/users.controller.ts
import { Roles } from '../common/decorators/roles.decorator';
import { RolesGuard } from '../common/guards/roles.guard';

@Controller('users')
@UseGuards(AuthGuard, RolesGuard)
export class UsersController {
  // ...

  @Delete(':id')
  @Roles('admin')
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

## 9. Interceptors

Interceptor ใช้สำหรับการจัดการ request และ response ก่อนและหลังการทำงานของ handler

### การสร้าง Logging Interceptor

สร้าง interceptor สำหรับการบันทึกเวลาที่ใช้ในการทำงาน:

```typescript
// src/common/interceptors/logging.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const method = request.method;
    const url = request.url;
    
    console.log(`[${method}] ${url} - ${new Date().toISOString()}`);
    
    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`[${method}] ${url} - ${Date.now() - now}ms`)),
      );
  }
}
```

### การใช้ Interceptor ใน Controller

ใช้ interceptor ใน controller:

```typescript
// src/users/users.controller.ts
import { Controller, UseInterceptors } from '@nestjs/common';
import { LoggingInterceptor } from '../common/interceptors/logging.interceptor';

@Controller('users')
@UseInterceptors(LoggingInterceptor) // ใช้ interceptor กับทุก route ใน controller
export class UsersController {
  // ...methods
}
```

### การสร้าง Response Transformation Interceptor

สร้าง interceptor สำหรับการแปลง response:

```typescript
// src/common/interceptors/transform.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
  statusCode: number;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(
      map(data => ({
        data,
        statusCode: context.switchToHttp().getResponse().statusCode,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

ใช้ TransformInterceptor ในระดับ global:

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
import { TransformInterceptor } from './common/interceptors/transform.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ /* options */ }));
  app.useGlobalInterceptors(new TransformInterceptor());
  await app.listen(3000);
}
bootstrap();
```

## 10. Middleware

Middleware ใช้สำหรับการจัดการ request ก่อนที่จะส่งไปยัง route handler

### การสร้าง Logging Middleware

สร้าง middleware สำหรับการบันทึก request:

```typescript
// src/common/middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`Request... Method: ${req.method}, URL: ${req.url}`);
    next();
  }
}
```

### การใช้ Middleware ใน Module

ใช้ middleware ใน module:

```typescript
// src/app.module.ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { UsersModule } from './users/users.module';

@Module({
  imports: [UsersModule, /* other modules */],
  // ...
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('users'); // ใช้กับ routes ที่ขึ้นต้นด้วย 'users'
  }
}
```

### การสร้าง Functional Middleware

นอกจาก class middleware แล้ว ยังสามารถสร้าง functional middleware ได้:

```typescript
// src/common/middleware/logger.middleware.ts
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request... Method: ${req.method}, URL: ${req.url}`);
  next();
}
```

ใช้ functional middleware:

```typescript
// src/app.module.ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { logger } from './common/middleware/logger.middleware';

@Module({
  // ...
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(logger)
      .forRoutes('*'); // ใช้กับทุก route
  }
}
```

## 11. Exception Filters

Exception Filter ใช้สำหรับการจัดการข้อผิดพลาดที่เกิดขึ้นในแอปพลิเคชัน

### การสร้าง Global Exception Filter

สร้าง filter สำหรับการจัดการข้อผิดพลาดทั้งหมด:

```typescript
// src/common/filters/http-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    
    const status = 
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;
        
    const message = 
      exception instanceof HttpException
        ? exception.getResponse()
        : 'Internal server error';
        
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}
```

ใช้ Exception Filter ในระดับ global:

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
import { AllExceptionsFilter } from './common/filters/http-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ /* options */ }));
  app.useGlobalFilters(new AllExceptionsFilter());
  await app.listen(3000);
}
bootstrap();
```

### การสร้าง Custom Exception

สร้าง custom exception:

```typescript
// src/common/exceptions/user-not-found.exception.ts
import { NotFoundException } from '@nestjs/common';

export class UserNotFoundException extends NotFoundException {
  constructor(userId: number) {
    super(`User with ID ${userId} not found`);
  }
}
```

ใช้ custom exception ใน service:

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { UserNotFoundException } from '../common/exceptions/user-not-found.exception';

@Injectable()
export class UsersService {
  // ...

  findOne(id: number) {
    const user = this.users.find(user => user.id === id);
    if (!user) {
      throw new UserNotFoundException(id);
    }
    return user;
  }
  
  // ...
}
```

## 12. การเชื่อมต่อฐานข้อมูล

NestJS สามารถเชื่อมต่อกับฐานข้อมูลได้หลายประเภท เราจะใช้ TypeORM ซึ่งเป็น ORM ที่นิยมใช้กับ TypeScript

### การติดตั้ง TypeORM

```bash
npm install @nestjs/typeorm typeorm pg
```

### การตั้งค่า TypeORM

สร้างไฟล์ configuration:

```typescript
// src/config/database.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export const databaseConfig: TypeOrmModuleOptions = {
  type: 'postgres',
  host: 'localhost',
  port: 5432,
  username: 'postgres',
  password: 'postgres',
  database: 'e_learning',
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  synchronize: true, // ไม่ควรใช้ในโหมด production
};
```

เพิ่ม TypeOrmModule ใน AppModule:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { databaseConfig } from './config/database.config';

@Module({
  imports: [
    TypeOrmModule.forRoot(databaseConfig),
    // other modules
  ],
  // ...
})
export class AppModule {}
```

### การสร้าง Entity

สร้าง entity สำหรับผู้ใช้งาน:

```typescript
// src/users/entities/user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### การใช้ Repository

เพิ่ม TypeOrmModule ใน UsersModule:

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

ใช้ Repository ใน UsersService:

```typescript
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { UserNotFoundException } from '../common/exceptions/user-not-found.exception';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const user = this.usersRepository.create(createUserDto);
    return this.usersRepository.save(user);
  }

  async findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  async findOne(id: number): Promise<User> {
    const user = await this.usersRepository.findOne({ where: { id } });
    if (!user) {
      throw new UserNotFoundException(id);
    }
    return user;
  }

  async update(id: number, updateUserDto: UpdateUserDto): Promise<User> {
    const user = await this.findOne(id);
    this.usersRepository.merge(user, updateUserDto);
    return this.usersRepository.save(user);
  }

  async remove(id: number): Promise<User> {
    const user = await this.findOne(id);
    return this.usersRepository.remove(user);
  }
}
```

## 13. การทำ Authentication และ Authorization

### การติดตั้ง Packages

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
```

### การสร้าง Auth Module

```bash
nest g module auth
nest g controller auth
nest g service auth
```

### การสร้าง JWT Strategy

```typescript
// src/auth/strategies/jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from '../constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, email: payload.email };
  }
}
```

### การสร้าง Auth Service

```typescript
// src/auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string): Promise<any> {
    const user = await this.usersService.findByEmail(email);
    if (user && await bcrypt.compare(password, user.password)) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { email: user.email, sub: user.id };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

### การสร้าง Auth Controller

```typescript
// src/auth/auth.controller.ts
import { Controller, Post, Body, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  async login(@Body() loginDto: LoginDto) {
    const user = await this.authService.validateUser(
      loginDto.email,
      loginDto.password,
    );
    
    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }
    
    return this.authService.login(user);
  }
}
```

### การตั้งค่า Auth Module

```typescript
// src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';
import { JwtStrategy } from './strategies/jwt.strategy';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '1h' },
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

### การสร้าง JWT Guard

```typescript
// src/auth/guards/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

### การใช้ JWT Guard ใน Controller

```typescript
// src/users/users.controller.ts
import { Controller, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';

@Controller('users')
@UseGuards(JwtAuthGuard) // ใช้ guard กับทุก route ใน controller
export class UsersController {
  // ...methods
}
```

## 14. การทำ Validation

เราได้เรียนรู้เกี่ยวกับการใช้ ValidationPipe และ class-validator ไปแล้ว ต่อไปเราจะเรียนรู้เกี่ยวกับการสร้าง custom validator

### การสร้าง Custom Validator

สร้าง custom validator สำหรับตรวจสอบว่าอีเมลไม่ซ้ำกัน:

```typescript
// src/common/validators/is-unique-email.validator.ts
import { registerDecorator, ValidationOptions, ValidatorConstraint, ValidatorConstraintInterface, ValidationArguments } from 'class-validator';
import { Injectable } from '@nestjs/common';
import { UsersService } from '../../users/users.service';

@Injectable()
@ValidatorConstraint({ async: true })
export class IsUniqueEmailConstraint implements ValidatorConstraintInterface {
  constructor(private usersService: UsersService) {}

  async validate(email: string, args: ValidationArguments) {
    const user = await this.usersService.findByEmail(email);
    return !user;
  }

  defaultMessage(args: ValidationArguments) {
    return `Email ${args.value} is already in use`;
  }
}

export function IsUniqueEmail(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsUniqueEmailConstraint,
    });
  };
}
```

ใช้ custom validator ใน DTO:

```typescript
// src/users/dto/create-user.dto.ts
import { IsEmail, IsNotEmpty, IsString, MinLength } from 'class-validator';
import { IsUniqueEmail } from '../../common/validators/is-unique-email.validator';

export class CreateUserDto {
  @IsEmail()
  @IsNotEmpty()
  @IsUniqueEmail()
  readonly email: string;

  // ...other fields
}
```
