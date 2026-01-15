步骤 1：数据库创建
打开 MySQL 客户端（Navicat），执行以下 SQL 创建专属数据库：

-- 创建数据库（指定utf8mb4字符集，兼容中文）
CREATE DATABASE IF NOT EXISTS library_system 
DEFAULT CHARACTER SET utf8mb4 
DEFAULT COLLATE utf8mb4_general_ci;

-- 选择目标数据库
USE library_system;
 
步骤 2：表结构创建
执行完整建表 SQL，创建 8 张核心表（含约束、外键、索引）：

-- 1. 图书类型字典表
CREATE TABLE `book_type_dict`  (
  `type_code` int(11) UNSIGNED NOT NULL COMMENT '类型编码',
  `type_name` varchar(50) NOT NULL COMMENT '类型名称',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`type_code`) USING BTREE
) ENGINE = InnoDB COMMENT = '图书类型字典表';

-- 2. 用户表
CREATE TABLE `user`  (
  `uid` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '用户编号',
  `user_name` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(64) NOT NULL COMMENT '密码（SHA256加密存储）',
  `role` tinyint(1) UNSIGNED NOT NULL COMMENT '角色(0-学生 1-管理员 2-审核员)',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`uid`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 3 COMMENT = '用户表';

-- 3. 学生表
CREATE TABLE `student`  (
  `sid` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '学生编号',
  `user_id` int(11) UNSIGNED NOT NULL COMMENT '用户编号',
  `student_no` varchar(20) NOT NULL COMMENT '学号',
  `student_name` varchar(20) NOT NULL COMMENT '学生姓名',
  `department` varchar(50) NOT NULL COMMENT '所属院系',
  `id_card_image` varchar(1024) NOT NULL DEFAULT '' COMMENT '学生证照片路径',
  `is_approved` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '借阅资格(0-未通过 1-通过)',
  `is_frozen` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '账号状态(0-正常 1-冻结)',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`sid`) USING BTREE,
  UNIQUE KEY `uk_student_no` (`student_no`) USING BTREE,
  INDEX `idx_student_name` (`student_name`) USING BTREE,
  CONSTRAINT `fk_student_user` FOREIGN KEY (`user_id`) REFERENCES `user` (`uid`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE = InnoDB COMMENT = '学生表';

-- 4. 图书表
CREATE TABLE `book`  (
  `bid` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '图书编号',
  `book_no` varchar(50) NOT NULL COMMENT '图书编码(A/B/C_书架号_编号)',
  `book_name` varchar(100) NOT NULL COMMENT '图书名称',
  `book_type` int(11) UNSIGNED NOT NULL COMMENT '类型编码(关联book_type_dict)',
  `author` varchar(50) NOT NULL COMMENT '作者',
  `description` varchar(2048) NOT NULL COMMENT '简介',
  `publisher` varchar(80) NOT NULL COMMENT '出版社',
  `publish_date` date NOT NULL COMMENT '出版时间',
  `cover_image` varchar(1024) NOT NULL DEFAULT '' COMMENT '封面路径',
  `status` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '状态(0-闲置 1-借阅中)',
  `borrow_count` int(11) UNSIGNED NOT NULL DEFAULT 0 COMMENT '借阅次数',
  `comment_count` int(11) UNSIGNED NOT NULL DEFAULT 0 COMMENT '评论次数',
  `is_recommend` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '新书推荐(0-否 1-是)',
  `create_user_id` int(11) UNSIGNED NOT NULL COMMENT '录入管理员ID',
  `modify_user_id` int(11) UNSIGNED NULL DEFAULT NULL COMMENT '修改管理员ID',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`bid`) USING BTREE,
  UNIQUE KEY `uk_book_no` (`book_no`) USING BTREE,
  INDEX `idx_book_name` (`book_name`) USING BTREE,
  CONSTRAINT `ck_book_no_format` CHECK (`book_no` REGEXP '^[ABC]_[0-9]+_[0-9]+$'),
  CONSTRAINT `fk_book_create_user` FOREIGN KEY (`create_user_id`) REFERENCES `user` (`uid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_book_modify_user` FOREIGN KEY (`modify_user_id`) REFERENCES `user` (`uid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_book_type` FOREIGN KEY (`book_type`) REFERENCES `book_type_dict` (`type_code`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE = InnoDB COMMENT = '图书表';

-- 5. 借阅证申请表
CREATE TABLE `borrow_apply`  (
  `ba_id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '申请编号',
  `student_id` int(11) UNSIGNED NOT NULL COMMENT '学生编号',
  `status` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '状态(0-待审核 1-通过 2-驳回)',
  `verify_time` datetime(0) NULL DEFAULT NULL COMMENT '审核时间',
  `verify_user_id` int(11) UNSIGNED NULL DEFAULT NULL COMMENT '审核员ID',
  `reject_reason` varchar(100) NULL DEFAULT '' COMMENT '驳回原因',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`ba_id`) USING BTREE,
  CONSTRAINT `fk_borrow_apply_student` FOREIGN KEY (`student_id`) REFERENCES `student` (`sid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_borrow_apply_verify_user` FOREIGN KEY (`verify_user_id`) REFERENCES `user` (`uid`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE = InnoDB COMMENT = '借阅证申请表';

-- 6. 图书评论表
CREATE TABLE `book_comment`  (
  `bc_id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '评论编号',
  `student_id` int(11) UNSIGNED NOT NULL COMMENT '学生编号',
  `book_id` int(11) UNSIGNED NOT NULL COMMENT '图书编号',
  `score` tinyint(1) UNSIGNED NOT NULL COMMENT '评分(1-5)',
  `comment` varchar(256) NOT NULL COMMENT '评论内容',
  `status` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '状态(0-待审核 1-通过 2-驳回)',
  `verify_time` datetime(0) NULL DEFAULT NULL COMMENT '审核时间',
  `verify_user_id` int(11) UNSIGNED NULL DEFAULT NULL COMMENT '审核管理员ID',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`bc_id`) USING BTREE,
  CONSTRAINT `fk_book_comment_student` FOREIGN KEY (`student_id`) REFERENCES `student` (`sid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_book_comment_book` FOREIGN KEY (`book_id`) REFERENCES `book` (`bid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_book_comment_verify_user` FOREIGN KEY (`verify_user_id`) REFERENCES `user` (`uid`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE = InnoDB COMMENT = '图书评论表';

-- 7. 图书借阅表（先创建基础表，后续补充字段）
CREATE TABLE `book_borrow`  (
  `bb_id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '借阅编号',
  `student_id` int(11) UNSIGNED NOT NULL COMMENT '学生编号',
  `book_id` int(11) UNSIGNED NOT NULL COMMENT '图书编号',
  `borrow_time` datetime(0) NOT NULL COMMENT '借阅时间',
  `status` tinyint(1) UNSIGNED NOT NULL DEFAULT 0 COMMENT '状态(0-待审核 1-通过 2-拒绝 3-已还)',
  `reject_reason` varchar(100) NULL DEFAULT '' COMMENT '拒绝原因',
  `verify_time` datetime(0) NULL DEFAULT NULL COMMENT '审核时间',
  `verify_user_id` int(11) UNSIGNED NULL DEFAULT NULL COMMENT '审核管理员ID',
  `return_time` datetime(0) NULL DEFAULT NULL COMMENT '归还时间',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`bb_id`) USING BTREE,
  CONSTRAINT `fk_book_borrow_student` FOREIGN KEY (`student_id`) REFERENCES `student` (`sid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_book_borrow_book` FOREIGN KEY (`book_id`) REFERENCES `book` (`bid`) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT `fk_book_borrow_verify_user` FOREIGN KEY (`verify_user_id`) REFERENCES `user` (`uid`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE = InnoDB COMMENT = '图书借阅表';

-- 8. 系统全局配置表
CREATE TABLE IF NOT EXISTS `sys_config` (
  `config_id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '配置编号',
  `config_key` varchar(50) NOT NULL COMMENT '配置键（唯一）',
  `config_value` int(11) NOT NULL COMMENT '配置值（数字类型，方便计算）',
  `config_desc` varchar(200) NOT NULL COMMENT '配置描述',
  `gmt_create` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `gmt_modified` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改时间',
  PRIMARY KEY (`config_id`) USING BTREE,
  UNIQUE KEY `uk_config_key` (`config_key`) USING BTREE
) ENGINE = InnoDB COMMENT = '系统全局配置表';

-- 9. 补充图书借阅表字段（借阅期限/应还日期）
ALTER TABLE `book_borrow`
ADD COLUMN `borrow_days` int(11) UNSIGNED NULL COMMENT '借阅期限（天），仅审核通过后赋值' AFTER `borrow_time`,
ADD COLUMN `due_date` datetime(0) NULL COMMENT '应还日期（=借阅通过时间+借阅期限，逾期判断核心）' AFTER `borrow_days`;
