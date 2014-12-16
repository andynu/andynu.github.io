---
layout: post
title: Vagrant
categories: devops
---

{% highlight ruby %}
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
end
{% endhighlight %}
