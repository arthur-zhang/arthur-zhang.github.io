---
layout: post
title:  "JavaMail API使用"
date:   2015-03-31 21:17:43
categories: java
---

####1. 集成maven,目前最新为1.5.2

     <dependency>
        <groupId>javax.mail</groupId>
        <artifactId>javax.mail-api</artifactId>
        <version>1.5.2</version>
    </dependency>
    <dependency>
        <groupId>com.sun.mail</groupId>
        <artifactId>javax.mail</artifactId>
        <version>1.5.2</version>
    </dependency>

最基本的发送代码如下：

    import javax.mail.Address;
    import javax.mail.Message;
    import javax.mail.Message.RecipientType;
    import javax.mail.MessagingException;
    import javax.mail.Session;
    import javax.mail.Transport;
    import javax.mail.internet.InternetAddress;
    import javax.mail.internet.MimeMessage;
    import java.io.UnsupportedEncodingException;
    import java.util.Properties;

    public class Main {
        public static void main(String[] args) {
            Properties props = new Properties();
            Session session = Session.getDefaultInstance(props);
            Message msg = new MimeMessage(session);

            Transport transport = null;

            try {
                Address brother18Address = new InternetAddress("xxoo@cvte.com", "Brother18");
                Address zhangyaAddress = new InternetAddress("zhangya@cvte.cn", "Arthur Zhang");

                msg.setFrom(brother18Address);
                msg.setRecipient(RecipientType.TO, zhangyaAddress);
                msg.setSubject("You must comply.");
                msg.setText("hahah");
                transport = session.getTransport("smtp");
                transport.connect("mail.cvte.com", 25, "zhangya@cvte.com", "xxx");
                transport.sendMessage(msg, msg.getAllRecipients());
            } catch (UnsupportedEncodingException | MessagingException e) {
                e.printStackTrace();
            } finally {
                if (transport != null) {
                    try {
                        transport.close();
                    } catch (MessagingException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
    
因为JavaMail API要兼容 java6，所以Transport没有实现AutoCloseable接口

另外send方法也可以使用Transport的静态接口`Transport.send()`

    import javax.mail.Address;
    import javax.mail.Message.RecipientType;
    import javax.mail.MessagingException;
    import javax.mail.Session;
    import javax.mail.Transport;
    import javax.mail.internet.InternetAddress;
    import javax.mail.internet.MimeMessage;
    import java.io.UnsupportedEncodingException;
    import java.util.Properties;

    public class Main {
        public static void main(String[] args) {
            Properties props = new Properties();
            props.put("mail.smtp.host", "smtp.163.com");
            props.put("mail.transport.protocol", "smtps");

            Session session = Session.getInstance(props);
            MimeMessage msg = new MimeMessage(session);

            try {
                Address brother18Address = new InternetAddress("zhangya_no1@163.com", "Brother18");
                Address zhangyaAddress = new InternetAddress("zhangya@cvte.cn", "Arthur Zhang");

                msg.setFrom(brother18Address);
                msg.setRecipient(RecipientType.TO, zhangyaAddress);
                msg.setSubject("You must comply.");
                msg.setText("hahah");
                Transport.send(msg, "zhangya_no1@163.com", "***");
            } catch (UnsupportedEncodingException | MessagingException e) {
                e.printStackTrace();
            }
        }
    }























