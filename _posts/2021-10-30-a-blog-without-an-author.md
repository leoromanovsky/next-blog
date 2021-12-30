---
layout: post
title:  "A Blog without an Author: TypeORM creates confusion with required Columns and optional Foreign Keys"
date: '2020-03-16T05:35:07.322Z'
categories: typeorm
author:
  name: Tim Neutkens
  picture: '/assets/blog/authors/tim.jpeg'
ogImage:
  url: '/assets/blog/hello-world/cover.jpg'
---

Storing data is one of the most critical choices we make when creating software and services.
We rely on our database schemas to enforce a service’s rules and a wrong column type or misconfiguration opens us up to subtle bugs, degrading our work in the eyes of customers.
Consider the case of a row with a foreign key but whose stored value is NULL - if accidental, the relationship between objects is forever lost.

From a customer’s perspective this manifests itself as storing an object, such as me hitting Publish on this blog, but later not being able to retrieve it.

## The problem with TypeORM models
The [most popular Javascript ORM in the Node ecosystem](https://typeorm.io/) surprised me recently with what I believe to be bad design choices.

Illustrating my findings interactively, below are two model definitions from the framework — I’d like you to guess what the table DDL will look like.
Pay particular attention to the required columns and foreign keys; scroll down for a deeper dive.

```
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
 
  @Column()
  firstName: string;
 
  @Column({ nullable: false })
  lastName: string;
 
  @Column({ nullable: true })
  birthDate: string;
 
  @Column({ default: true })
  isActive: boolean;
 
  @OneToMany(() => Post, (post) => post.user)
  posts: Post[];
}
 
@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;
 
  @Column({ nullable: false })
  title: string;
 
  @ManyToOne(() => User, {
    eager: true,
  })
  @JoinColumn({ name: 'user_id' })
  user: User;
}
```

The TypeORM migration program produces surprising output:

```
CREATE TABLE "user" (
  "id" SERIAL NOT NULL, 
  "first_name" character varying NOT NULL, # required!
  "last_name" character varying NOT NULL, 
  "birth_date" character varying, 
  "is_active" boolean NOT NULL DEFAULT true, 
  CONSTRAINT "PK_cace4a159ff9f2512dd42373760"
  PRIMARY KEY ("id")
)

CREATE TABLE "post" (
  "id" SERIAL NOT NULL, 
  "title" character varying NOT NULL, 
  "user_id" integer, # not required!
  CONSTRAINT "PK_be5fda3aac270b134ff9c21cdee"
  PRIMARY KEY ("id")
)
 
ALTER TABLE "post"
ADD CONSTRAINT "FK_52378a74ae3724bcab44036645b"
FOREIGN KEY ("user_id") 
REFERENCES "user"("id") ON DELETE NO ACTION ON UPDATE NO ACTION
```

* Did you expect `firstName` to be a required field? What about the model definition tipped you off?
* Does it make sense for a `Post` not to have a `User`?

## Unintuitive defaults lead to bugs

In the code above we saw unexpected results when relying on TypeORM’s default parameters. Consider this application code that we would produce from it.

### Missing foreign key
```
const us = UserService.find(1)
const post = PostFactory.create('title', 'content')
await PostService.save(post)
```

A critical bug created where the Post is disconnected from the User that owns it.
### Over-requirement of columns
The developer only specified lastName as a required field so why is the code failing to save?

```
const user = new User()
user.lastName = 'surname'
user.save() # fails!
```

The database will produce a failure to save the model since firstName is required as well.

### The best code should not include any comments describing its behavior
A quick scan suffices to intuit how it fits into the larger picture.
Nowhere is this more important than how data is stored and accessed by your application.

I don’t think this is a case of “Read the Freaking Manual” — After working in a codebase for an extended period of time we do learn its peculiarities. It would be acceptable if TypeORM picked a path but my biggest gripe is the inconsistent behavior between regular and join columns.

I think that the TypeORM project made mistakes with their API; alas it has been downloaded millions of times. Is there any way it can be reversed?

For a tool we can’t control I’m going to focus on working with it — learning to mitigate its quirks.

## Ruby on Rails gets it right

I used to work heavily in this framework and left feeling jaded. Looking back I recognize the hallmarks of a mature and well considered tool.

### Model definitions only include required fields
```
# app/model/product.rb
class Product < ActiveRecord::Base
  validates :flag, :presence => true
end
```

From looking just at the model, we can’t immediately tell what optional columns exist but we sure know which are necessary to save it. Rails expresses itself with brevity.

### Columns are nullable by default
```
# migration ORM
create_table(:objects, primary_key: 'guid') do |t|
  t.column :name, :string, limit: 80
end
# generated DDL
CREATE TABLE objects (
  guid bigint auto_increment PRIMARY KEY,
  name varchar(80)
)
Foreign keys are not nullable by default
# migration ORM
create_join_table :products, :categories do |t|
  t.index :product_id
  t.index :category_id
end
# generated DDL
CREATE TABLE products_categories (
  product_id int NOT NULL,
  category_id int NOT NULL,
)
```

While Rails also exhibits inconsistent behavior I believe the defaults are more inline with what a developer would have picked anyways. In particular requiring the values of foreign keys to be populated avoids broken relationship between models.

## Moving forward with TypeORM

I’ve been exploring Prisma and look forward to writing up my experiences in a future post.
For now I plan on explicitly calling out nullable on all my `@Column` and `@JoinTable` TypeORM decorators to make clear to myself and peers the intended usage of each attribute.

```
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
 
  @Column({ nullable: false })
  firstName: string;
 
  @Column({ nullable: false })
  lastName: string;
 
  @Column({ nullable: true })
  birthDate: string;
 
  @Column({ default: true })
  isActive: boolean;
 
  @OneToMany(() => Post, (post) => post.user)
  posts: Post[];
}
 
@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;
 
  @Column({ nullable: false })
  title: string;
 
  @ManyToOne(() => User, {
    eager: true,
    nullable: false,
  })
  @JoinColumn({ name: 'user_id' })
  user: User;
}
```

## Wrapping Up
I was reminded this week that while I enjoy delivering quickly and not always sweating the details, our computer code is also a means of communication between current and future peers.
As your team and product grows, consider the APIs you create less as short-term necessities but as guardrails around acceptable and repeated usage.
