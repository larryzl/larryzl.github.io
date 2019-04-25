---
layout: post
title: "Python 模块之 smtplib"
date: 2018-09-29 15:45:11 +0800
category: Python
layout: post
tags: [Python]
---
* content
{:toc}

> 需求：  
> Python 通过 smtplib 向多人发送邮件

修改 脚本中发件用户名、密码、发件服务器地址等信息，可以直接调用方法发送邮件  

收件人是多个，`mail_to`参数传入列表

发送附件，`file`填写文件绝对路径

	!/usr/bin/env python
	
	import smtplib
	import os
	
	def sendMail(mail_to,alert_info,title,file=None):
	    '''
	    :param mail_to: 收件人列表 type: list
	    :param alert_info: 邮件内容 type: str
	    :param file:  附件文件名
	    :param title: 邮件标题
	    :return: 
	    '''
	    
	    
	    from email.mime.multipart import MIMEMultipart
	    from email.mime.text import MIMEText
	    from email.mime.application import MIMEApplication
	    
	    if not isinstance(mail_to,list):mail_to = [mail_to]
	    username = "teste@xxx.com"
	    password = "xxxxx"
	    sender = username
	    receivers = ",".join(mail_to)
	
	    msg = MIMEMultipart()
	    msg['Subject'] = title
	    msg['From'] = sender
	    msg['To'] = receivers
	    msg["Accept-Language"]="zh-CN"
	    msg["Accept-Charset"]="ISO-8859-1,utf-8"
	
	    puretext = MIMEText(alert_info,format,'utf-8')
	    msg.attach(puretext)
	    
	    if file:
	        xlsxpart = MIMEApplication(open(file, 'rb').read())
	        xlsxpart.add_header('Content-Disposition', 'attachment', filename='%s' % os.path.basename(file))
	        msg.attach(xlsxpart)
	
	    try:
	        client = smtplib.SMTP()
	        # 发件SMTP 服务器地址
	        client.connect('mail.xxxx.com')
	        client.login(username, password)
	        client.sendmail(sender, receivers.split(","), msg.as_string())
	        client.quit()
	        print('mail send ok')
	    except smtplib.SMTPRecipientsRefused:
	        print('Recipient refused')
	    except smtplib.SMTPAuthenticationError:
	        print('Auth error')
	    except smtplib.SMTPSenderRefused:
	        print('Sender refused')
	    except smtplib.SMTPException as e:
	        print(e.message)
