# Azure Local 2503 展開方法

## 0. 本手順が想定しているネットワーク構成図
<details>

![image](https://github.com/osamut/AzureStackHCI_23H2/assets/1791583/8c009dd9-d86e-4931-bcb7-a60e1534df88)
</details>

## 1. Azure Local の要件を満たすサーバーや NIC、スイッチなどのハードウェアの準備
<details>

- 要件を確実に満たすため、専門家に相談しつつ、Azure Local カタログからハードウェアを選択する
  -  https://azurestackhcisolutions.azure.microsoft.com/#/catalog
- サーバーのスペックはサイジングツールで概要を把握し、ベンダーと細かな要件を詰めながら調整する
  -　 https://azurestackhcisolutions.azure.microsoft.com/#/sizer 
- その他の要件も確認し、準備に利用する
  - [物理ネットワーク要件](https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/physical-network-requirements?tabs=overview%2C23H2reqs)
  - [ホストネットワーク要件](https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/host-network-requirements)
  - [ファイアウォール要件](https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/firewall-requirements)
- 導入するハードウェアの Azure Stack HCI セキュリティ対応状況の確認
  - Azure Stack HCI 展開時に[各種セキュリティ設定](https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/security-features)の有効・無効を聞かれるため事前に確認
  - 導入するハードウェアがすべて対応していることが望ましい　 ※特にTPM の有無
- (オプション) 環境チェッカーツールを利用した環境の事前評価
  - [展開中にも行われるため必須ではないが、作業を開始する前に事前チェックも可能](https://learn.microsoft.com/ja-jp/azure-stack/hci/manage/use-environment-checker?tabs=connectivity)
</details>

## 2. Azure Stack HCI ハードウェア環境設定とOSインストール
<details>
	
- Azure Stack HCI ハードウェアセットアップと ToR スイッチ設定
	- 環境に合わせて VLAN なども設定しておく
	- 今回の構成はこのような状態
- Azure ポータルにアクセスし、検索ボックスに [Azure Stack HCI] と入力、[Azure Stack HCI 管理画面] を表示する
- Azure ポータルの Azure Stack HCI 管理画面から Azure Stack HCI OS の英語版の ISO イメージをダウンロード　～英語版のみの提供になりました～
- IDRAC/ILO など物理サーバー管理ツールのコンソールにて各ノードにアクセスし、Azure Stack HCI OS ISO をマウント
- OS のインストール画面は Windows Server とほぼ同じなので迷うことはないはず
- 物理サーバー管理ツールにて ISO イメージをアンマウントすること
	- マウントしたままだと Azure Stack HCI 展開中の BitLocker 暗号化の画面で進まなくなることがわかっている
</details>
    
## 3. Azure Stack HCI OS インストール後の設定 ※各ノードにて実施
<details>
	
### 1: OSインストール後にパスワード設定画面が出てくるので、12文字以上の複雑なパスワードを入力 ※Sconfig の画面へ遷移
### 2: SConfig での設定
- 8の [Network settings] にて管理用の NIC に IP アドレスを設定
	- DHCP から IP をもらっている複数の NIC がある場合、番号の小さい IP アドレスを管理用にするとよさそう
	- [静的IPアドレス] [サブネットマスク] [デフォルトゲートウェイ] の設定後、[DNS サーバー] を追加設定
- 7 の [Remote desktop] にてリモートデスクトップを Enabled に変更
	- リモートデスクトップだとコピー＆ペーストが容易で、作業の生産性が上がるため
	- Azure Stack HCI 展開後は自動で Disable にしてくれる
- 9 の [Date and time] にて [Internet Time] タブを開き NTP サーバーと同期できていることを確認 ※ここ大事！！！！
	- デフォルトは time.windows.com だが変更も可能
 	- 通信できない場合は社内のタイムサーバーと同期する必要あり　--環境内に設置したNTPサーバーと同期させる場合は、ソースとなるNTPサーバーの時刻が正しいことを確認
 	- 特に仮想マシンをNTPサーバーにする場合は、物理ホスト側の時刻を正確な時刻に合わせておく必要あり  
	- Azure Stack HCI OSはデフォルトのタイムゾーンが Pacific Timeになっているので時刻のズレに気づかない可能性あり　－－JSTに変更するなどして確実に確認すること
- 4 の [Remote management] にて ping を有効にすると、通信確認などに便利
- 2 の [Computer name] にてコンピュータ名を変更し、再起動

### 3: NIC の名前設定や DHCP 無効化などを行う
- リモートデスクトップ mstsc.exe にて各ノードにリモートアクセス
	- 管理者名は コンピュータ名￥administrator 　パスワードはインストール後に設定したものを利用
-  現時点での NIC の状態を確認
```
Get-NetIPAddress -AddressFamily IPv4 | select InterfaceAlias,IPAddress,PrefixOrigin
```
```
--結果例--
InterfaceAlias　　　　　　　 　IPAddress　　　　PrefixOrigin
 --------------　　　　 　　　　---------　　　　　------------
Port1			      10.29.146.4　　 　　Dhcp　　　　　・・・管理＋VM 通信用に利用するセカンダリNIC
Port2　　　　　　　　　　　　 　10.29.146.13　　　  Manual　　　　・・・手動で IP 設定した管理用 NIC　管理＋VM 通信用に利用
Port3　　　　　　	　　　169.254.222.78　　  WellKnown　　 ・・・Software Defined Storage 用の RDMA NIC１
Port4		　　　　　　　 169.254.123.122　　 WellKnown　　 ・・・Software Defined Storage 用の RDMA NIC２
Ethernet　　　　　　　　　　　　169.254.1.2　　　　 Dhcp　　　　　・・・サーバーの USB とホストをつなぐために利用
Loopback Pseudo-Interface 1　  127.0.0.1　　　　　 WellKnown     ・・・今回は気にしなくてよい        
--------
```
		
- Azure Stack HCI の Network ATC (インテントベースのネットワーク管理)で利用するため、環境に合わせて NIC 名を変更
	- 今回の環境で使う NIC 名が Port1,Port2,Port3,Port4 になっていることが前提で書いてある
 	- 既存環境の NIC 名を使って正しく設定する必要あり
- 最初に手動で IP アドレス設定をした管理用 NIC の名前を MGMT_VM1 に変更
```
Rename-NetAdapter -Name "Port1" -NewName "MGMT_VM1"
```
 - それ以外の 3 つの NIC の名前変更
```
Rename-NetAdapter -Name "Port2" -NewName "MGMT_VM2"
Rename-NetAdapter -Name "Port3" -NewName "Storage1"
Rename-NetAdapter -Name "Port4" -NewName "Storage2"
```
- NIC の DHCP を無効化
```
Get-NetAdapter -Name "MGMT_VM1" | Set-NetIPInterface -Dhcp Disabled
Get-NetAdapter -Name "MGMT_VM2" | Set-NetIPInterface -Dhcp Disabled
Get-NetAdapter -Name "Storage1" | Set-NetIPInterface -Dhcp Disabled
Get-NetAdapter -Name "Storage2" | Set-NetIPInterface -Dhcp Disabled
```
- NIC ドライバーをインストール
	- サーバーベンダーのサイトからダウンロードした最新のサポートされた NIC ドライバーをインストール
	- 管理用マシンなどでダウンロードしたドライバーを含むフォルダーを共有しておき、Azure Stack HCI ノードから「net use v: \\コンピュータ名\共有名」などで接続、ドライバーのインストールを行う
	- ドライバーのセットアップ exe を起動すると Azure Stack HCI OS 上でも GUI が表示され、インストールが可能だった
    - インストール終了後、「net use v: /delete」などでマウントを解除しておく
- 各ノードで以下のコマンドを実行し、NIC に OS 標準のドライバー(Inbox Driver ＝ DriverProvider に Microsoft)が残っていないことを確認する
```
Get-NetAdapter -Name * | Select *Driver*
```

__※ Ethernet = Ethernet Remote NDIS Compatible Device という Inbox Driver NIC が存在する可能性あり　・・・対処が必要__
- [Enternet Remote NDIS Compatible Device という NIC について](https://www.dell.com/support/kbdoc/ja-jp/000130077/poweredge-idrac-%E3%83%80%E3%82%A4%E3%83%AC%E3%82%AF%E3%83%88-%E6%A9%9F%E8%83%BD-%E3%81%AE-%E4%BD%BF%E7%94%A8-%E6%96%B9%E6%B3%95)

- 対処方法１：以下のコマンドにて該当するクラスター検証時のエラーを回避
```
# Exclude iRDAC USB NIC from cluster validation
New-Item -Path HKLM:\system\currentcontrolset\services\clussvc
New-Item -Path HKLM:\system\currentcontrolset\services\clussvc\parameters
New-ItemProperty -Path HKLM:\system\currentcontrolset\services\clussvc\parameters -Name ExcludeAdaptersByDescription  -Value "Remote NDIS Compatible Device"
```
- 対処方法２：以下のコマンドにて該当するプラグ＆プレイデバイスを削除
	- pnputil /enum-devices /ids /class net コマンドにて Network Device の Instance ID を確認可能
	- プラグ＆プレイデバイスのためノード再起動後に Enternet Remote NDIS Compatible Device は自動復活する
```
pnputil /remove-device "USB\VID_413C&PID_A102\5678"
```

- IPv6 の無効化
```
Disable-NetAdapterBinding -Name * -ComponentID ms_tcpip6
```
- VLAN 構成と NIC の関係を確認しておく
	- Software Defined Storage 用の RDMA NIC には VLAN 設定が必須
	- Azure Stack HCI 展開時にVLAN 強制適用で、VLAN 0 も不可
	- よって、ストレージ用 NIC がスイッチ経由でつながっている場合はスイッチ側の VLAN 設定も行い、どの NIC と結線されているかを理解しておく
	- クラスター作成後の NIC の VLAN 設定確認は Get-NetAdapter -Name * | fl にて可能

- 各ノードに対して Hyper-V を有効化
```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```
- 再起動
- 再起動後 リモートデスクトップで再接続し、以下を実行して Enternet Remote NDIS Compatible Device を無効化しておくと安心
```
pnputil /remove-device "USB\VID_413C&PID_A102\5678"
```
</details>

## 4. Azure Stack HCI 展開用の Active Directry 事前設定
<details>
	
**※ Active Directry に管理者としてアクセスできるマシンであればどこからでも実施可能**
- Active Directory に作成する OU 名と新規追加する展開用のユーザー名、パスワードを決める
	- 既存 OU 名の指定、既存ユーザー名の入力も可能だが、以下のようにAzure Stack HCI に最適化されることになる
	- OU にはホストやクラスターオブジェクトが追加され、サーバーボリュームが暗号化されている場合、OU を削除すると BitLocker 回復キーも削除されるため、専用の OU が望ましい
	- 処理中に入力する展開用のユーザー ID も、HCI 展開用の権限設定が自動的に行われるため専用 ユーザー ID のほうが望ましい
	- 展開用ユーザーのパスワードは12 文字以上で、小文字、大文字、数字、特殊文字を含む必要あり
- ツールのインストール
```
Install-Module AsHciADArtifactsPreCreationTool -Repository PSGallery -Force
```
-  作成する OU 名を OU=xx,DC=xxx,DC=xxx という形式で $NewOU に代入
```
$NewOU = "OU=xx,DC=xxx,DC=xxx"
```
- Active Directory に新規 OU と展開用のユーザーID を作成
- __以下のコマンドを実行すると、ユーザー名とパスワードを入力する画面がポップアップしてくるので、事前に決めた情報を入力__
```
New-HciAdObjectsPreCreation -AzureStackLCMUserCredential (Get-Credential) -AsHciOUName $NewOU
```
-  [Active Directory ユーザーとコンピュータ] ツールにて 新しい OU と展開用のユーザーができていることを確認
</details>
	
## 5. Azureポータルでの事前作業
<details>
	
- Azure ポータル(https://portal.azure.com) にログオン
- Azure ポータルを英語に変更
	- 不要なトラブルを回避するため
 	- (本手順書作成時は日本語ポータルだと仮想スイッチ名の文字化けなどに遭遇)
- Azure Stack HCI に関連するオブジェクトを登録するリソースグループを新規作成
	- (リソースグループに対して各オブジェクトが作成される)
	- 2024年5月4日時点で Japan East での作業が可能になっている
	- リソースグループに対して、Azure 側で作業をするアカウントに以下の管理権限を付与
 		- Azure Connected Machine Onboarding
   		- Azure Connected Machine Resource Administrator
	- サブスクリプションに以下のリソースプロバイダーが登録されていることを確認し、登録されていなければ登録する		
		- Microsoft.HybridCompute
  		- Microsoft.GuestConfiguration
		- Microsoft.HybridConnectivity
  		- Microsoft.AzureStackHCI
</details>

## 6. Azure Stack HCI 各ノードを Azure Arc で Azure と接続
<details>
	
### 各 Azure Stack HCI ノードを Azure Arc に登録するための手順１　モジュールのインストール
- PSGallery を信頼できるリポジトリとして登録 　・・・入力を求められたら Y を入力し処理を継続
```
Register-PSRepository -Default -InstallationPolicy Trusted
```
- その他 必要な PowerShell モジュールのインストール
```
Install-Module Az.Accounts -RequiredVersion 3.0.1
Install-Module Az.Resources -RequiredVersion 6.12.0
Install-Module Az.ConnectedMachine -RequiredVersion 0.8.0
```

- Azure Arc 登録用のモジュールをインストール　・・・入力を求められたら All の A を入力
```
Install-Module AzsHCI.ARCinstaller
```

### 各 Azure Stack HCI ノードを Azure Arc に登録するための手順２　登録に必要な情報の入力と収集
- Azure サブスクリプション ID を指定　・・・Azure ポータルのサブスクリプション管理画面から入手可能
```
$Subscription = "利用するサブスクリプションID"
```
- リソースグループ名を指定
```
$RG = "利用するリソースグループ名"
```
- テナント ID を指定　・・・ Azure ポータルの Microsoft Entra ID 管理画面から入手可能
```
$Tenant = "利用するテナントID"
```
- [サポートされている Azure リージョン](https://learn.microsoft.com/en-us/azure-stack/hci/concepts/system-requirements-23h2#azure-requirements)を指定 (Japan East も可能になりました)
```
$Region = "Japan East"
```
- デバイスコードを使ってサブスクリプションにログオン認証を実施
	- Azure Stack HCI が Server Core ベースでログオンのポップアップが出せないため、他のマシンのブラウザーで https://login.microsoftonline.com/common/oauth2/deviceauth にアクセスし、Azure Stack HCI 画面に表示されたコードを使って Azure ユーザーをログオンさせている
```
Connect-AzAccount -SubscriptionId $Subscription -TenantId $Tenant -DeviceCode
```

- Azure Stack HCI Docsに記載はないが、ArmToken取得にエラーが発生する場合には以下のコマンドを実行 （エラーが出て再実行する際は　Connect-AzAccountからやり直したほうが良いかも）
```
Update-AzConfig -EnableLoginByWam $false
```

- ログオンした Azure ユーザーのトークンを取得
```
$ARMtoken = (Get-AzAccessToken).Token
```
- Arc に登録するためのアカウントの ID を取得
```
$id = (Get-AzContext).Account.Id
```

### 各 Azure Stack HCI ノードを Azure Arc に登録するための手順３　実際の登録作業
- 上記で入力、取得した情報を使って Azure Arc に登録
```
Invoke-AzStackHciArcInitialization -SubscriptionID $Subscription -ResourceGroup $RG -TenantID $Tenant -Region $Region -Cloud "AzureCloud" -ArmAccessToken $ARMtoken -AccountID $id
```
- Azure Stack HCI の画面で登録完了を確認
- Azure ポータルの [Azure Arc] - [Machines] にて、登録作業を行った Azure Stack HCI マシン名をクリック
- Azure ポータルの Azure Arc マシン管理画面にて [Extensions] をクリックし、以下の 4 つの拡張機能の追加が成功するのを待つ
	- AzureEdgeDeviceManagement
	- AzureEdgeLifecycleManager
	- AzureEdgeRemoteSupport
	- AzureEdgeTelemetryAndDiagnostics
- すべての拡張機能のインストールが完了すると、Arc への登録も完了
	- しばらく放置しておくと 5 つ目 (MDE.Windows) の拡張モジュールが増えている

- (オプション) Azure Arc 登録されたノードを削除する方法
	- Azure Stack HCI クラスター展開がうまく進まなかったり、Azure から Azure Stack HCI クラスターを削除した場合、Azure Arc 管理画面から Azure Stack HCI ノードも削除されることになる
	- しかし、Azure Stack HCI 各ノードには既に設定された情報が残っているため、以下のコマンドを使って削除を行う
```
$Subscription = "利用するサブスクリプションID"
$RG = "利用するリソースグループ名"
$Tenant = "利用するテナントID"
Connect-AzAccount -SubscriptionId $Subscription -TenantId $Tenant -DeviceCode
$ARMtoken = (Get-AzAccessToken).Token
$id = (Get-AzContext).Account.Id
Remove-AzStackHciArcInitialization -SubscriptionID $Subscription -ResourceGroup $RG -TenantID $Tenant -Cloud "AzureCloud" -ArmAccessToken $ARMtoken -AccountID $id
```
</details>

## 7. Azure Stack HCI クラスター展開
<details>
	
### 事前設定
- Azure ポータルが英語であることを確認  ・・・不要なトラブルを回避するため
- サブスクリプションに対し、Azure 側の作業をするアカウントに以下の管理権限を付与
	- Azure Stack HCI Administrator
  	- Reader
 - リソースグループに対し、Azure 側の作業をするアカウントに以下の管理権限を付与
 	- Key Vault Data Access Administrator
  	- Key Vault Secrets Officer　　　      (日本語ポータル作業時は ”キーコンテナーシークレット責任者” を探す)
   	- Key Vault Contributor
   	- Storage Account Contributor
### クラスター構築作業
-  Azure ポータルの [Azure Arc] - [Azure Stack HCI] 管理画面にて、[All Clusters (PREVIEW)] を選択
	-  PREVIEW ではない画面にしたい場合は、画面内の [Old Experience] をクリックすると GA 済みの画面が表示される
#### 1. [+Create] メニューから [Azure Stack HCI Cluster] を選択
#### 2. Basics タブ
- 2-1: 展開に利用する [サブスクリプション] と [リソースグループ] を選択
	- リソースグループが違うと画面一番下に Arc に登録したサーバー一覧が表示されないので注意
- 2-2: [Cluster name] を入力
- 2-3: Region はサポートしているリージョンを入力　※ Japan East で OK
- 2-4: Key vault name では [Create a new key vault] をクリックし、右に出てくる画面で [Create] をクリック
	- 権限付与のリンクが表示されたらクリック 
	- 繰り返し同じ作業をした場合は既存の Key Vault を削除するか、Key Vault name を変更する事で対応
	- Key Vault は削除しても削除済みリストに残るので、削除済みリストからさらに削除する必要がある
 - 2-5: リソースグループ内の Azuer Stack HCI ノードの一覧が表示されるので、クラスターに参加させるノードにチェックを入れし、[Validate selected servers] をクリック
 	- このタイミングでホストのNIC情報を取得するため不要なNICの削除などはこの前に行っておくこと
 - 2-6: Validate が完了したら [Next: Configuration] をクリック
#### 3. Configuration タブ
 - [New configuration] が選択されていることを確認し [Next: Networking] をクリック
 	- テンプレートが用意できている場合はテンプレートを利用
#### 4.  Networking タブ
- ※ ここは実際の環境に合わせて設定をする必要がある
- ※ 以下は NIC4 枚の環境にて、管理＆VM 用ネットワークに NICx2 (最初に IP アドレスを設定した NIC を含む)、ストレージ用に NICx2 (VLAN 設定必須) を利用する想定
- 4-1: [Network switch for storage] をクリック
- 4-2: [Group management and compute traffic] をクリック
- 4-3: インテント名「Compute_Management」に対して [MGMT_VM1] を選択
- 4-4: [+ Select another adapter for this traffic] をクリックして [MGMT_VM2] を追加
- 4-5: [Customize network settings] をクリックして「RDMA Protocol」を Disabled に変更
- 4-6: インテント名「Storage」に対して　[Storage1] を選択
- 4-7: 必須項目となっている VLAN ID には[環境に合わせたVLAN ID] を入力
   	- ノード間の通信で利用するためのもの
   	- スイッチレスやスイッチ側ですべての VLAN ID の通信を許可していればデフォルトのままでOK
   	- スイッチ側で設定がされていればその VLAN ID を間違わずに入力すること
- 4-8: [+ Select another adapter for this traffic] をクリックして [Storage2] 追加
- 4-9: [環境に合わせた VLAN ID] を入力
- 4-10: [Customize network settings] をクリックして「RDMA Protocol」を環境に合わせる　(RoCEv2/iWarp など)
- 4-11: Azure Stack HCI が利用する最低 7 つの IP アドレス範囲を用意し、[Starting IP] ~ [Ending IP] として入力
- 4-12: [サブネットマスク　例 255.255.255.0] を入力
- 4-13: [デフォルトゲートウェイのIPアドレス] を入力
- 4-14: [DNS Server のIPアドレス] を入力
- 4-15: [Next: Management] をクリック
#### 5. Management タブ
- 5-1: Azure から Azure Stack HCI クラスターに指示を出す際に利用するロケーション名として [任意のCustom location name] を入力
   	- 良く使うので、プロジェクト名や場所、フロアなどを使って、わかりやすい名前を付けておくこと
	- 思い浮かばない時はクラスター名に-cl とつけておくとわかりやすいかも
- 5-2: Cloud witness 用に [Create new]を クリック、さらに右に出てきた内容を確認
	- 必要に応じて修正のうえ、[Create] をクリックし、Azure Storage Account を作成
- 5-3: [ドメイン名 例 contoso.com] を入力
- 5-4: [OU名 　例 OU=test,DC=contoso,DC=com ] を入力　　　※Active Directory の準備の際に設定したOU
- 5-5: Deployment 用の [Username] を入力　　※ Active Directory の準備の際に指定した Deployment 用のユーザー名
- 5-6: [Password]  [Confirm password] を間違えないように入力
- 5-7: Local administrator としての [Username] を入力　　※特別な設定をしていなければ Administrator で OK
- 5-8: [Password]  [Confirm password] を間違えないように入力
- 5-9: [Next: Security] をクリック
#### 6. Security タブ
- [Recommended security settings] が選択されていることを確認し [Next: Advanced] をクリック
	- 古いサーバーを使って展開をする場合など、推奨設定の機能を満たせない場合は [Custommized security settings] をクリックして有効にしたい項目のみを選択
#### 7. Advanced タブ
- [Create workoad volumes and required infrastructure volumes] が選択されていることを確認し[Next: Tags] をクリック
	- 既定で、Software Defined Storage プールに Infrastructure ボリュームと、Azure Stack HCI 各ノードを Owner とする論理ボリュームを自動作成してくれる
#### 8. Azure 上のオブジェクトを管理しやすくする任意のタグをつけ、[Next: Validation] をクリック
#### 9. Validation タブ
- 9-1: 特に問題が無ければ Resource Creation として6つの処理を行うため全て Succeeded になることを確認
- 9-2: [Start Validation] をクリック
- 9-3: 更に 11 個のチェックが行われ Validation が完了したら [Next: Preview + Create] をクリック
#### 10. [Create] をクリックし、Azure Stack HCI Cluster Deployment を開始
   - 画面が Azure Stack HCI 管理画面の ”Settings” にある「Deployments」が選択された状態に遷移するので [Refresh] をクリックして状況を確認できる
   - 手元の 2 ノードで 2 時間半程度かかった
   - "Deploy Arc infrastructure components" ステップでエラーが出る場合 (HCIノードへの接続を繰り返し行いタイムアウト)、Failover Cluster Manager画面の自動作成されたResource Bridge VM のネットワーク設定にて、「Enable MAC address spoofing」を有効にし、「Protected network」を無効にすることでエラー回避可能
   - OS の更新やドメイン参加を含め Azure Stack HCI 23H2 クラスター作成作業が自動で行われ、終了すると Azure から管理可能な状態になる
   - 途中エラーが出た場合はログを確認するなどして対処し [Rerun deployment] を実施


</details>


<!--
__[Azure Stack HCI 23H2 展開後の作業はこちら](/toCreateVMs.md)__
-->
