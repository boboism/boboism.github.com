---
layout: post
title: "如何使用Ruby来写WebService的Provider"
date: 2013-05-22 12:14
comments: true
categories: [SOA, WebService, API, Ruby]
---

最近在做一个程序，用于我的EAM（固定资产管理系统）跟公司的ERP对接固定资产的数据。开始是想直接用MES在用的接口系统，修修补补上。
但是发现这个接口系统相对来说耦合比较厉害，如果要改来用，怕且都要搞个半个月才能上线。
后来就打算直接用Ruby写算了。于是就有了这篇文章。
下面看看其中的一个XML规格(大概),这是由ERP项目组定义的规法：

``` xml 资产导入
<?xml version="1.0" encoding="UTF-8"?>
<Interface Sender="EAM" Receiver="ERP" Billtype="F10">
  <Bill>
    <BillHeader>
      <AcceptanceDate>2006-08-28 00:00:00</AcceptanceDate>
      <AssetName>笔记本电脑</AssetName>
      <AssetNo>703021</AssetNo>
      <Brand>IBM X60</Brand>
      <Category>7</Category>
      <IsSpecialFund>否</IsSpecialFund>
      <IsTariff>否</IsTariff>
      <IsVat>否</IsVat>
      <Model></Model>
      <OriginalValue>100.0</OriginalValue>
      <Salvage>20.0</Salvage>
      <SerialNo>L3B6534</SerialNo>
      <ServiceDate>2006-08-28 00:00:00</ServiceDate>
      <SubCategory>03</SubCategory>
      <TaxPreferType>不属于</TaxPreferType>
    </BillHeader>
    <BillBody>
      <Entry>
        <AllocationQuantity>0.5</AllocationQuantity>
        <ConstructionPeriod>1</ConstructionPeriod>
        <CostCenter>00001</CostCenter>
        <ManagementDepartment>00002</ManagementDepartment>
        <SpecialPurpose>01</SpecialPurpose>
      </Entry>
      <Entry>
        <AllocationQuantity>0.5</AllocationQuantity>
        <ConstructionPeriod>1</ConstructionPeriod>
        <CostCenter>00008</CostCenter>
        <ManagementDepartment>00002</ManagementDepartment>
        <SpecialPurpose>01</SpecialPurpose>
      </Entry>
    </BillBody>
  </Bill>
</Interface>
```

我最近都在用Ruby on Rails在开发项目（个人／公司），所以感觉上这个应该没有太大困难。不过还是有以下问题需要解决：

* 不太规范的SOAP XML内容格式。
* How to call webservice？
* 定时执行方式？

想用ActiveRecord直接生成XML，但是需求的XML跟ActiveRecord直接生成出来的样子不太一样。后来看了ActiveRecord的XML相关的builder的代码，觉得还是使用builder定制吧。
就有了以下的代码：

``` ruby ActiveRecord::Base#to_erp_xml
module ActiveRecord
  class Base
    def to_erp_xml(options={})
      require 'builder' unless defined?(Builder)
      options = options.dup
      xml = ::Builder::XmlMarkup.new
      xml.instruct! :xml, :version => "1.0", :encoding => "UTF-8"
      interface_attributes = [:sender, :receiver, :billtype].inject({}) do |acc, im| 
        acc.merge({im.to_s.camelize => options[:interface][im]}) if options[:interface][im]
      end
      xml.Interface(interface_attributes) do |interface|
        if options[:include]
          interface.Bill do |bill|
            bill.BillHeader do |bill_header|
              self.class.column_names.sort.each do |col|
                unless options[:except] && Array(options[:except]).include?(col.to_sym)
                  bill_header.tag! col.to_s.camelize, self[col.to_sym]
                end
              end
            end
            bill.BillBody do |bill_body|
              Array(options[:include]).each do |attr|
                self.send(attr.to_sym).each do |association|
                  bill_body.Entry do |entity|
                    association.class.column_names.sort.each do |col| 
                      unless options[:except] && Array(options[:except]).include?(col.to_sym)
                        entity.tag! col.to_s.camelize, association[col.to_sym]
                      end
                    end
                  end
                end
              end
            end
          end
        end
      end
      xml.target!.to_s
    end
  end
end
```

为啥写的这么别扭？其实是因为我直接使用了db的view去抓数据。哈哈哈，为防以后会有变更，这样直接修改SQL就可以了：

``` ruby AssetF10&AllocationF10
class AssetF10 < ActiveRecord::Base
  self.table_name = "vw_asset_erp"
  self.primary_key = "id"

  has_many :allocations, :class_name => "AllocationF10", :foreign_key => "asset_id"
end

class AllocationF10 < ActiveRecord::Base
  self.table_name = "vw_asset_allocation_erp"
  self.primary_key = "id"

  belongs_to :asset, :class_name => "AssetF10"#, :foreign_key => "asset_id"
end
```

到这步，基本上XML是能够生成了，但是如何去调用呢？上rubygems.org看了看包的使用量，soap这一块savon相对来说比较好，而且文档还是很充分。感觉这样写的话相对Java的SOA调用使用xfire非常简洁方便。

``` ruby Savon.client
client = Savon.client(:wsdl => "http://172.18.81.20/BOI/Service.asmx?WSDL")
ws_attrs = {
  :interface => {:sender => "EAM", :receiver => "ERP", :billtype => "F10"},
  :func_name => "FixedAssetImport",
  :handshake => "111111"
}
receivable_assets = []
unreceivable_assets = []
AssetF10.all.each do |asset|
  token = UUIDTools::UUID.random_create.to_s.gsub("-","").upcase
  parameters = asset.to_erp_xml(ws_attrs.merge({:include => :allocations, :except => [:id, :asset_sync_status, :asset_id]}))
  resp = client.call(:boi_invoke, :message => {
    :from       => "#{ws_attrs[:interface][:sender]}#{ws_attrs[:handshake]}", 
    :to         => ws_attrs[:interface][:receiver], 
    :token      => token,
    :func_name  => "#{ws_attrs[:func_name]}_#{ws_attrs[:interface][:billtype]}", 
    :parameters => "#{parameters.to_s}" })
  p parameters 
  if resp.body[:boi_invoke_response][:boi_invoke_result]
    receivable_assets << {asset.asset_no => "TOKEN=#{token}"}
    # update records
    asset.asset_sync_status = 1
    asset.save
  else
    unreceivable_assets << {asset.asset_no => "TOKEN=#{token} #{resp.body[:boi_invoke_response][:result]}"}
  end
end
```
OK，webservice call的问题已经解决了。就剩下最后一个问题，定时调用！
一开始就想通过linux crontab来定时调用这接口，这样就会相对于固定资产管理系统本身的sidekiq独立起来（毕竟这个是一个月才调用1次的接口，无需实时调要求，感觉也没有这个必要）

``` bash eam_soa_f10.sh
#!/bin/sh
/home/it/.rvm/rubies/ruby-1.9.3-p374/bin/ruby /home/it/projects/ruby/cli/eam_soa/eam_soa_f10.rb >> /home/it/projects/ruby/cli/eam_soa/eam_soa_f10.log 2>&1
```

接着在crontab中增加一行调用：

``` bash crontab
* * 25 * * /home/it/projects/ruby/cli/eam_soa/eam_soa_f10.sh
```

OK~ 大功告成。相对Java版本的接口系统，同样功能大概只用了其1个类（差不多）的代码量就完成了～。

