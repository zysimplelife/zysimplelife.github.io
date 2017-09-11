---
layout: post
title:  "Script in Micro Service Architecture"
date: 2017-09-11 15:10:00 +0000
categories: Micro Service
---

### Introduction
Using Java in Micro-Service Architecture sounds like a bad practice, especially if it is without any framework like spring boot, but that is what we have done so far. In our case, each service need a start-up script to handle life cycle. After fixing several issues in more than one year, we finally got a so called stable version.     

### Architecture 
It is not a contained based architecture, so a custom-made framework to control services has been mad. Each service should provide a script to handle life cycle and health check. I think the is only version 1, it should be changed in container environment. 

### Issues we had faced 

#### Checking if Process has been started 

Checking if process has been started is a very important part of start-up script, or more than one same service could be started and cause unpredictable issue. In order to solve this problem, we create a PID file after service starting. Each time this script is executed, it will check if the PID file is existed.  

Beside checking the PID file, script should also check if the PID in the PID file is really existed by "ps" command. If failed to find the pid, this service should be consider to down rather than started.
 

#### Timeout 
Services may start in a dead loop, which cause the start script hang forever.  In order to solve this problem, each script should check java process in a given timeout


#### Execution in Parallel
After introducing timeout check in script, start-up script will wait until the service is ready. If another one try to execute this script, the PID file could be overwrite. This may cause another  unpredictable issue. In order to solve this problem, we changed the script again. Now every script will cache the PID in both parameter and PID file. So, withing this executing, it will only use the value in parameter.
 


### Start script

![Start up script ](https://www.planttext.com/plantuml/img/XLN9Rjim4BthAzwjDf06Q7kBKJI15y1Tq21npmKjZIp24YcGb9U_xt2nOjangeF1PPOtRzxGZzO9uxgcpZ8dNKru3ViMhxcHhOlRZtZAdTn9TyHCYeqH3R8i2vvPzZ2jAD_YUJd3GjOqoMI9aT_HGLf7nRSnN4KAeoFPSGOaUxPwDZedQq-64xuClkbu7eylt3b0m4G5lc9bEl9kL5l2IEahWuNWcs2X2bbc0xjhisYKAUq8HcugnsPB1MqACd0QZTWIR6VuXynEvWIfy5tiH5zA9IpMn71jZ7t74LmbBMoaKn4LreVA0mbhxQh0NCdCGQYY3yI1NuzSVdCF3miU4tFk-KcmBtYnJhU3-XumhKaexfYXt6bpd8J35shqZjkyUdfPpMT_5ykVymd0LuEgCJ20hKHTSs783GbFsVeOwsuJN5qYU9ru4QLZoagffAsasaGwkZOs8XO3bDfiOyEYp8PKSiWKRHACBl3vRs2_bn5YIELpCeMjC0oSwWZk8ZN4HaYL04m5ToIqmdTX6ihiDUtDRu73JNG-fCVXep2MqEk-ppjXf5ZpnclAedVYdYQjU9O9hRPMUZlM31qex_nverc2hXAgTzCgYf4BDbpl_LN1mBxokKUhvQ_vxp-hCLYK_SG41e8Y1xf8Tq5qvtA8z5nFPb20rs5S3gVoD8DfbU3qlhByx-2_m_7PAv8BzS75xhCMUEWKd6n8T-uEUQqPYdJlqE9B_-6bawGywKzVo7L6OHM3uDFdBpTRFJU65PBhWjv6HTHoem8IbFnYqjvBRf0VQNj1KvyvHhsTbOk_9ShmfA9JT6Xv-ECx53_1WIgUJJg7hBiwg2szn1SmJxlKlDHXVnqcEBsQsICl-8_a7m00)

[https://www.planttext.com/?text=XLN9Rjim4BthAzwjDf06Q7kBKJI15y1Tq21npmKjZIp24YcGb9U_xt2nOjangeF1PPOtRzxGZzO9uxgcpZ8dNKru3ViMhxcHhOlRZtZAdTn9TyHCYeqH3R8i2vvPzZ2jAD_YUJd3GjOqoMI9aT_HGLf7nRSnN4KAeoFPSGOaUxPwDZedQq-64xuClkbu7eylt3b0m4G5lc9bEl9kL5l2IEahWuNWcs2X2bbc0xjhisYKAUq8HcugnsPB1MqACd0QZTWIR6VuXynEvWIfy5tiH5zA9IpMn71jZ7t74LmbBMoaKn4LreVA0mbhxQh0NCdCGQYY3yI1NuzSVdCF3miU4tFk-KcmBtYnJhU3-XumhKaexfYXt6bpd8J35shqZjkyUdfPpMT_5ykVymd0LuEgCJ20hKHTSs783GbFsVeOwsuJN5qYU9ru4QLZoagffAsasaGwkZOs8XO3bDfiOyEYp8PKSiWKRHACBl3vRs2_bn5YIELpCeMjC0oSwWZk8ZN4HaYL04m5ToIqmdTX6ihiDUtDRu73JNG-fCVXep2MqEk-ppjXf5ZpnclAedVYdYQjU9O9hRPMUZlM31qex_nverc2hXAgTzCgYf4BDbpl_LN1mBxokKUhvQ_vxp-hCLYK_SG41e8Y1xf8Tq5qvtA8z5nFPb20rs5S3gVoD8DfbU3qlhByx-2_m_7PAv8BzS75xhCMUEWKd6n8T-uEUQqPYdJlqE9B_-6bawGywKzVo7L6OHM3uDFdBpTRFJU65PBhWjv6HTHoem8IbFnYqjvBRf0VQNj1KvyvHhsTbOk_9ShmfA9JT6Xv-ECx53_1WIgUJJg7hBiwg2szn1SmJxlKlDHXVnqcEBsQsICl-8_a7m00](https://www.planttext.com/?text=XLN9Rjim4BthAzwjDf06Q7kBKJI15y1Tq21npmKjZIp24YcGb9U_xt2nOjangeF1PPOtRzxGZzO9uxgcpZ8dNKru3ViMhxcHhOlRZtZAdTn9TyHCYeqH3R8i2vvPzZ2jAD_YUJd3GjOqoMI9aT_HGLf7nRSnN4KAeoFPSGOaUxPwDZedQq-64xuClkbu7eylt3b0m4G5lc9bEl9kL5l2IEahWuNWcs2X2bbc0xjhisYKAUq8HcugnsPB1MqACd0QZTWIR6VuXynEvWIfy5tiH5zA9IpMn71jZ7t74LmbBMoaKn4LreVA0mbhxQh0NCdCGQYY3yI1NuzSVdCF3miU4tFk-KcmBtYnJhU3-XumhKaexfYXt6bpd8J35shqZjkyUdfPpMT_5ykVymd0LuEgCJ20hKHTSs783GbFsVeOwsuJN5qYU9ru4QLZoagffAsasaGwkZOs8XO3bDfiOyEYp8PKSiWKRHACBl3vRs2_bn5YIELpCeMjC0oSwWZk8ZN4HaYL04m5ToIqmdTX6ihiDUtDRu73JNG-fCVXep2MqEk-ppjXf5ZpnclAedVYdYQjU9O9hRPMUZlM31qex_nverc2hXAgTzCgYf4BDbpl_LN1mBxokKUhvQ_vxp-hCLYK_SG41e8Y1xf8Tq5qvtA8z5nFPb20rs5S3gVoD8DfbU3qlhByx-2_m_7PAv8BzS75xhCMUEWKd6n8T-uEUQqPYdJlqE9B_-6bawGywKzVo7L6OHM3uDFdBpTRFJU65PBhWjv6HTHoem8IbFnYqjvBRf0VQNj1KvyvHhsTbOk_9ShmfA9JT6Xv-ECx53_1WIgUJJg7hBiwg2szn1SmJxlKlDHXVnqcEBsQsICl-8_a7m00 "Edit")

### Stop script 

![Stop Script](https://www.planttext.com/plantuml/img/fP91Jm8n48Nl_HNl28dw0oH6Y422UY0kTrDsPzcHtNRJ3aJ-lRFTXIXwyz9qfczUtdpfD8eDScXgOuIb9cIfRf7bWLlforlCSk4ZombpjhjW6nXZqgGnzqyLvNkiLtCikQQ9uHAZhg9FZaB5unXIaSFeH75iW46lgdNmESLu5axqCSqExKNVlXfNWvI92ZnW4mxKZL4T2IFdVmcMLb-ImXLScX-wtx9UP9mNGk1T9IfREVXGK81uD7PFY8UW1uKZvmHsUBP7UrcbiX5RqhYnzxvH1wau8lOu7L4HEwiyGTXwgHAvKid1kk9YfCRPITTlxj35GfT9cNTyXjZNM14QP9lPssOnVr-kNXUJSrFzlpN-JV-5wneVtTBj8FNbXSVVgAFuzWmttSrKA_rqNm00)
[https://www.planttext.com/?text=fP91Jm8n48Nl_HNl28dw0oH6Y422UY0kTrDsPzcHtNRJ3aJ-lRFTXIXwyz9qfczUtdpfD8eDScXgOuIb9cIfRf7bWLlforlCSk4ZombpjhjW6nXZqgGnzqyLvNkiLtCikQQ9uHAZhg9FZaB5unXIaSFeH75iW46lgdNmESLu5axqCSqExKNVlXfNWvI92ZnW4mxKZL4T2IFdVmcMLb-ImXLScX-wtx9UP9mNGk1T9IfREVXGK81uD7PFY8UW1uKZvmHsUBP7UrcbiX5RqhYnzxvH1wau8lOu7L4HEwiyGTXwgHAvKid1kk9YfCRPITTlxj35GfT9cNTyXjZNM14QP9lPssOnVr-kNXUJSrFzlpN-JV-5wneVtTBj8FNbXSVVgAFuzWmttSrKA_rqNm00](https://www.planttext.com/?text=fP91Jm8n48Nl_HNl28dw0oH6Y422UY0kTrDsPzcHtNRJ3aJ-lRFTXIXwyz9qfczUtdpfD8eDScXgOuIb9cIfRf7bWLlforlCSk4ZombpjhjW6nXZqgGnzqyLvNkiLtCikQQ9uHAZhg9FZaB5unXIaSFeH75iW46lgdNmESLu5axqCSqExKNVlXfNWvI92ZnW4mxKZL4T2IFdVmcMLb-ImXLScX-wtx9UP9mNGk1T9IfREVXGK81uD7PFY8UW1uKZvmHsUBP7UrcbiX5RqhYnzxvH1wau8lOu7L4HEwiyGTXwgHAvKid1kk9YfCRPITTlxj35GfT9cNTyXjZNM14QP9lPssOnVr-kNXUJSrFzlpN-JV-5wneVtTBj8FNbXSVVgAFuzWmttSrKA_rqNm00 "Edit")


### Improvement Point

- This Script is not container based
- Health check shall be better than STARTED_FILE, Maybe a JMX Command 



