# Terraform Note

- 快速部屬大量資源
- 快速部屬一模一樣的環境
- 可以版控工具整合，做過的變更都可以有紀錄
- 概念可先參考微軟以下影片
{%youtube zFCdsq_G9Rk %}
{%youtube KGEYUQL5PyQ %}

----
## 檔案類型
terraform的檔案類型為tf檔，可以只使用一個檔案來控制，也可以使用以下4個常見的檔案來控制部屬
`providers.tf` 用來控制資源提供商
`main.tf` 主要程式碼，定義出需要布建的資源
`variables.tf` 共用的變數檔
`outputs.tf` 部屬完成後要顯示的資訊


----
## 語法格式
```=
resource "<資源類型>" "<變數>" {
  name     = "<在Azure上顯示的資源名稱>"
  location = "<區域>"
}
```
----
## 基本指令

初始化terraform環境
```=
terraform init
```

檢查程式碼是否有錯誤的語法格式，並產出一份計畫，可以用來檢查是否和預期的結果相同
```=
terraform plan
```

開始部屬
```=
terraform apply
```
----
## 建立虛擬機器的範例
設定資源提供者，範例會先將程式碼寫在一個檔案，並以Azure為主
```=
provider "azurerm" {
  features {}
}
```
以下範例為建立名稱為RG1的資源群組，地區設定為日本東部
```=
resource "azurerm_resource_group" "example" {
  name     = "RG1"
  location = "japaneast"
}
```
後續建立資源時可以直接建在資源群組所在區域
```=
  location = azurerm_resource_group.example.location
  resource_group_name  = azurerm_resource_group.example.name
```
建立資源群組A
```=
resource "azurerm_resource_group" "rga" {
  name     = "tf_rga"
  location = "eastasia"
}
```
建立虛擬網路77
```=
resource "azurerm_virtual_network" "vnet77" {
  name                = "terra-virtual-network77"
  address_space       = ["10.77.0.0/16"]
  location            = azurerm_resource_group.rga.location
  resource_group_name = azurerm_resource_group.rga.name
}
```
建立虛擬網路子網路77-1
```=
resource "azurerm_subnet" "subnet77-1" {
  name                 = "subnet77-1"
  resource_group_name  = azurerm_resource_group.rga.name
  //與vnet77關聯
  virtual_network_name = azurerm_virtual_network.vnet77.name    
  
  address_prefixes      = ["10.77.1.0/24"]
}
```
建立網路介面卡
```=
resource "azurerm_network_interface" "nic_A" {
  name                = "nic-A"
  location            = azurerm_resource_group.rga.location
  resource_group_name = azurerm_resource_group.rga.name

  ip_configuration {
    name                          = "ipconfig"
    //與subnet77-1關聯
    subnet_id                     = azurerm_subnet.subnet77-1.id
    private_ip_address_allocation = "Dynamic"
  }
}
```
建立windows虛擬機器
```=
resource "azurerm_windows_virtual_machine" "windowsvm1" {
  name                 = "windows1"
  resource_group_name = azurerm_resource_group.rga.name
  location            = azurerm_resource_group.rga.location
  size                 = "Standard_DS1_v2"
  admin_username       = "adminuser"
  admin_password       = "Admin@123456"
  network_interface_ids = azurerm_network_interface.nic_A.id]

  os_disk {
	name                 = "osdisk-A}"
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2016-Datacenter"
    version   = "latest"
  }
}
```

----

## 開始部屬
將建立虛擬機器的程式碼包成一個檔案如:`VM.tf`，便可以開始進行部屬
- 首先到Azure Portal開啟cloudshell，並切換為powershell模式
- 檢查目前的設定用帳戶，如不是預期佈署的訂用帳戶，記得先切換[訂用帳戶](https://hackmd.io/@NormanMD/Hy3101yPh)
- cloudshell預設都有安裝terraform，因此可以直接開始初始化

```=
terraform init
```

- 檢查程式碼是否有錯誤的語法格式，並產出一份計畫，可以用來檢查是否和預期的結果相同
```=
terraform plan
```

- 開始部屬
```=
terraform apply
```
- 等待部屬完成後，即可以開始使用部屬出來的資源

- 資源使用完畢後，可以使用以下指令刪除此次作業部建的所有資源
```=
terraform destroy
```

----
## 變更資源設定
部屬完畢後的資源，若是想做變更，直接修改程式碼即可

以變更虛擬機器大小為例，只需要修改機器大小這一行後重新執行部屬的動作
```=
size                 = "Standard_DS1_v2"    //修改規格
```
根據目前測試，Azure Portal上能做的變更都能做得到

----
## 執行過的測試
[terraform測試項目.xlsx](https://systexgroup-my.sharepoint.com/:x:/g/personal/2200008_systexsoftware_com_tw/EQcfrjgrkS1PruSm9B95yVgBNdPIKDMETNv1t3CmrFHbAA?e=6KsX8W)

持續測試中

| 編號 |測試項目 |	測試內容 |	測試結果 |
|----|----------|---------|----------|
|1  |VM |建立VM(linux) |OK |	
|2	|VM	|建立VM(windows) |OK |
|3	|VM	|同時建立十台虛擬機器A1~A10(windows、linux) |OK |
|4	|VM	|修改VM大小 |OK |
|5	|VM	|修改VM網卡綁定的虛擬網路 |<span style="color:red">NO</span> |
|6	|VM	|修改VM網卡綁定的子網路(在同一個虛擬網路下) |OK |
|7	|網路	|建立刪除虛擬網路及子網路 |OK |
|8	|網路	|編輯虛擬網路及子網路區段 |OK |
|9	|網路	|建立刪除NSG及NSG規則 |OK |
|10	|網路	|修改NSG規則 |OK |
|11	|網路	|網路介面卡關聯NSG、子網路關聯NSG |OK |
|12	|網路	|建立公用IP與網路介面卡關聯 |OK |
|13	|網路	|虛擬網路對等互連 | |
|14	|AVD |建立桌面集區 | |
|15	|AVD |建立工作階段主機(multi session) | |
|16	|AVD |建立工作階段主機(custom image) | |
|17	|AVD |指派使用者 | |
|18	|儲存體 |建立儲存體帳戶 | |
|19	|儲存體 |建立檔案共用 | |
|20	|其他	|切換訂用帳戶佈屬 | |
|21	|其他	|同時佈屬兩個不同的tf檔 | |
|22	|其他	|Azure Policy設定 | |
|23	|其他	|RBAC設定 | |
|24	|AOAI | 部屬| |
|25	|AVS | 部屬| |
|26 |Azure firewall | 部屬| |





----
## 後續
- 與Azure Devops結合，透過git版控工具連結，修改後的程式碼會被推送到pipelines進行自動化測試，與Azure Devops的結合會用另一個版面說明

----
## 參考資料

[【🔥 #15MIN雲端熱知識 🔥 EP26】用怦然心動的 IaC 魔法變出你想要的資源吧！（Terraform 篇） － 上](https://www.youtube.com/watch?v=zFCdsq_G9Rk)
[【🔥 #15MIN雲端熱知識 🔥 EP27】用怦然心動的 IaC 魔法變出你想要的資源吧！(Terraform 篇) － 下](https://www.youtube.com/watch?v=KGEYUQL5PyQ)
[azurerm](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)


https://ithelp.ithome.com.tw/articles/10233759
https://ithelp.ithome.com.tw/articles/10258904
