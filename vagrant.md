## 基础命令
```sh
# 初始化 Vagrantfile
vagrant init centos/7
vagrant up
vagrant ssh

vagrant status
vagrant halt
vagrant destroy
```

## 端口转发
单个端口
```ruby
config.vm.network "forwarded_port", guest: 80, host: 80
```
多个端口
```ruby
for i in 62001..62050
  config.vm.network "forwarded_port", guest: i, host: i
end
```

## 分配 ip
```ruby
config.vm.network "public_network"
```

## 共享目录
```ruby
config.vm.synced_folder "../data", "/data"
```

## 插件使用
```sh
vagrant plugin list
vagrant plugin install vagrant-scp
vagrant scp ../labs/ node01:/home/vagrant/labs/
```

## ssh 登录
vagrant ssh-config >> ~/.ssh/config
