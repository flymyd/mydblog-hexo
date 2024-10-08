---
title: Nest.js基础搭建与MySQL单表CRUD的操作
toc: true
date: 2023-04-12 17:24:04
tags: [Nest.js, TypeScript, Node.js]
categories:
  - [前端技术, 工程搭建]
---
## 安装相关依赖

```shell
yarn global add @nestjs/cli
nest new mydblog-server --strict && cd mydblog-server
yarn add typeorm @nestjs/typeorm mysql
```

## 配置数据源

src/config/Database.ts

```typescript
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export const AppDataSource: TypeOrmModuleOptions = {
  type: 'mysql',  //别的也行
  host: '数据库IP',
  port: 数据库端口号,
  username: '用户名',
  password: '密码',
  database: '待连接的数据库',
  synchronize: true,  //同步表结构
  logging: true,  //打印日志
  autoLoadEntities: true, //自动扫描实体
};
```

src/app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AppDataSource } from './config/Database';

@Module({
  imports: [TypeOrmModule.forRoot(AppDataSource)],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## 新建CRUD模块

[TypeORM中对于Entity的定义](https://typeorm.io/entities)

```shell
nest g res user   //res后跟的是模块名
- REST API
- Y
```

src/user/entities/user.entity.ts

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from "typeorm"

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number

  @Column({
    length: 100,
  })
  name: string

  @Column("text")
  address: string

  @Column()
  nickname: string

  @Column("double")
  stature: number
}
```

src/user/user.module.ts

```typescript
import { Module } from '@nestjs/common';
import { UserService } from './user.service';
import { UserController } from './user.controller';
import { TypeOrmModule } from "@nestjs/typeorm";
import { User } from "./entities/user.entity";

@Module({
  imports: [TypeOrmModule.forFeature([User])],  //这里引入
  controllers: [UserController],
  providers: [UserService]
})
export class UserModule {}
```

src/user/dto/create-user.dto.ts

```typescript
export class CreateUserDto {
  name: string;
  nickname: string;
  address: string;
}
```

src/user/user.service.ts

```typescript
import { Injectable } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { InjectRepository } from '@nestjs/typeorm';
import { User } from './entities/user.entity';
import { Repository } from 'typeorm';

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User) private userRepository: Repository<User>, //注入操作类
  ) {}

  async create(createUserDto: CreateUserDto) {
    createUserDto.address = '野兽邸';
    return await this.userRepository.save(createUserDto);
  }

  async findAll() {
    return await this.userRepository.find();
  }

  async findOne(id: number) {
    return await this.userRepository.findOneBy({ id });
  }

  async update(id: number, updateUserDto: UpdateUserDto) {
    //和创建同理
    return `This action updates a #${id} user`;
  }

  async remove(id: number) {
    return await this.userRepository.delete(id);
  }
}
```

用Postman测试一下，成功！
