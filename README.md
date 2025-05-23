# Azure Local 2503 展開方法

## 0. 本手順が想定しているネットワーク構成図
<details>

![image](https://github.com/osamut/AzureStackHCI_23H2/assets/1791583/8c009dd9-d86e-4931-bcb7-a60e1534df88)

> [!NOTE]
>- あくまでもテスト用ですが、各手順のスクリーンショットによる解説はこちらにあります！
>- https://github.com/osamut/AzureLocal_2503_Deploy/blob/main/AzureLocal_%E5%B1%95%E9%96%8B-%E7%89%A9%E7%90%86.pdf

</details>

## 1. Azure Local の要件を満たすサーバーや NIC、スイッチなどのハードウェアの準備
<details>

- 要件を確実に満たすため、専門家に相談しつつ、Azure Local カタログからハードウェアを選択する
  -  https://azurelocalsolutions.azure.microsoft.com/#/catalog
  - サーバーのスペックはサイジングツールで概要を把握し、ベンダーと細かな要件を詰めながら調整する
  -　 https://azurelocalsolutions.azure.microsoft.com/#/sizer
  - その他の要件も確認し、準備に利用する
  - [物理ネットワーク要件](https://learn.microsoft.com/ja-jp/azure/azure-local/concepts/physical-network-requirements?view=azloc-2503&tabs=overview%2C23H2reqs)
  - [ホストネットワーク要件](https://learn.microsoft.com/ja-jp/azure/azure-local/concepts/host-network-requirements?view=azloc-2503)
  - [ファイアウォール要件](https://learn.microsoft.com/ja-jp/azure/azure-local/concepts/firewall-requirements?view=azloc-2503)
- 導入するハードウェアの Azure Local セキュリティ対応状況の確認
  - Azure Local 展開時に[各種セキュリティ設定](https://learn.microsoft.com/ja-jp/azure/azure-local/concepts/security-features?view=azloc-2503)の有効・無効を聞かれるため事前に確認
  - 導入するハードウェアがすべて対応していることが望ましい　 ※特にTPM の有無
- (オプション) 環境チェッカーツールを利用した環境の事前評価
  - [展開中にも行われるため必須ではないが、作業を開始する前に事前チェックも可能](https://learn.microsoft.com/ja-jp/azure/azure-local/manage/use-environment-checker?view=azloc-2503&tabs=connectivity)
</details>

## 2. Azure Local ハードウェア環境設定とOSインストール
<details>
	
- Azure Local ハードウェアセットアップと ToR スイッチ設定
	- 環境に合わせて VLAN なども設定しておく
	- 今回の構成はこのような状態
- Azure ポータルにアクセスし、検索ボックスに [Azure Local] と入力、[Azure Local 管理画面] を表示する
- Azure ポータルの Azure Local 管理画面から Azure Local OS の英語版の ISO イメージをダウンロード　～英語版のみの提供になりました～
- IDRAC/ILO など物理サーバー管理ツールのコンソールにて各ノードにアクセスし、Azure Local OS ISO をマウント
- OS のインストール画面は Windows Server とほぼ同じなので迷うことはないはず
- 物理サーバー管理ツールにて ISO イメージをアンマウントすること
	- マウントしたままだと Azure Local 展開中の BitLocker 暗号化の画面で進まなくなることがわかっている
</details>
    
## 3. Azure Local OS インストール後の設定 ※各ノードにて実施
<details>
	
### 1: OSインストール後にパスワード設定画面が出てくるので、12文字以上の複雑なパスワードを入力 → Sconfig の画面へ自動遷移
> [!CAUTION]
> ### 現在 ”Azure Local” の SConfig ではNICのIPアドレス設定が反映されないことがある。その場合、PowerShell にて実施(以下にサンプルあり)

### 2: [15]で一旦 Sconfig の画面を終了 → PowerShell の画面へ自動遷移
-  現時点でリンクアップしている NIC のリストを確認
```
Get-NetAdapter * | Where-Object {$_.status -eq "Up"} | select name,InterfaceDescription,Status

```
- ノードのAzure Arc登録や管理用に使うNICの名前(”port1"とか”SLOT 3 Port 1"とか)を控える
```
Get-NetAdapter -Name “port3” | New-NetIPAddress -AddressFamily IPv4 -IPAddress 10.100.50.11 -PrefixLength 24 -DefaultGateway 10.100.50.1
Get-NetAdapter "port3" | Set-DnsClientServerAddress -ServerAddresses 10.100.50.10
```
### 2: SConfig での設定
- 7 の [Remote desktop] にてリモートデスクトップを Enabled に変更
	- リモートデスクトップだとコピー＆ペーストが容易で、作業の生産性が上がるため
	- Azure Local 展開後は自動で Disable にしてくれる
- 9 の [Date and time] にて [Internet Time] タブを開き NTP サーバーと同期できていることを確認 ※ここ大事！！！！
	- デフォルトは time.windows.com だが変更も可能
 	- 通信できない場合は社内のタイムサーバーと同期する必要あり　--環境内に設置したNTPサーバーと同期させる場合は、ソースとなるNTPサーバーの時刻が正しいことを確認
 	- 特に仮想マシンをNTPサーバーにする場合は、物理ホスト側の時刻を正確な時刻に合わせておく必要あり  
	- Azure Local OSはデフォルトのタイムゾーンが Pacific Timeになっているので時刻のズレに気づかない可能性あり　－－JSTに変更するなどして確実に確認すること
- 2 の [Computer name] にてコンピュータ名を変更し、再起動

### 3: NIC 設定や DHCP 無効化などを行う
- リモートデスクトップ mstsc.exe にて各ノードにリモートアクセス
	- 管理者名は コンピュータ名￥administrator 　パスワードはインストール後に設定したものを利用
-  現時点での NIC の状態を確認
```
Get-NetAdapter * | Where-Object {$_.status -eq "Up"} | select name,InterfaceDescription,Status
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
		
- Azure Local の Network ATC (インテントベースのネットワーク管理)で利用するため、環境に合わせて NIC 名を変更
	- 本記事では NIC 名が Port1,Port2,Port3,Port4 になっていることが前提で書いてある
 	- 複数ノードで物理NICとOSから見たNIC名の割り当てに違いがあれば、統一しておく (その際に利用するNIC名変更コマンドは以下)
```
Rename-NetAdapter -Name "Port1" -NewName "MGMT_VM1"
```
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
	- 管理用マシンなどでダウンロードしたドライバーを含むフォルダーを共有しておき、Azure Local ノードから「net use v: \\コンピュータ名\共有名」などで接続、ドライバーのインストールを行う
	- ドライバーのセットアップ exe を起動すると Azure Local OS 上でも GUI が表示され、インストールが可能だった
    - インストール終了後、「net use v: /delete」などでマウントを解除しておく
- 各ノードで以下のコマンドを実行し、NIC に OS 標準のドライバー(Inbox Driver ＝ DriverProvider に Microsoft)が残っていないことを確認する
```
Get-NetAdapter -Name * | Select *Driver*
```

__※ Ethernet = Ethernet Remote NDIS Compatible Device という Inbox Driver NIC が存在する可能性あり　・・・対処は不要になった__

- VLAN 構成と NIC の関係を確認しておく
	- Software Defined Storage 用の RDMA NIC には VLAN 設定が必須
	- Azure Local 展開時に VLAN を強制適用するため、VLAN 0 は不可
	- よって、ストレージ用 NIC がスイッチ経由でつながっている場合はスイッチ側の VLAN 設定も行い、どの NIC と結線されているかを理解しておく
	- 今回の環境は Storage1 は VLAN 147、Storage2 は VLAN 148 に接続されている
	- クラスター作成後の NIC の VLAN 設定確認は Get-NetAdapter -Name * | fl にて可能

</details>

## 4. Azure Local 展開用の Active Directry 事前設定
<details>
	
**※ Active Directry に管理者としてアクセスできるマシンであればどこからでも実施可能**
https://learn.microsoft.com/ja-jp/azure/azure-local/deploy/deployment-prep-active-directory?view=azloc-2503

- Active Directory に作成する OU 名と新規追加する展開用のユーザー名、パスワードを決める
	- 既存 OU 名の指定、既存ユーザー名の入力も可能だが、以下のように Azure Local に最適化されることになる
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
- サブスクリプションに以下のリソースプロバイダーが登録されていることを確認し、登録されていなければ登録する		
	- Microsoft.HybridCompute
	- Microsoft.GuestConfiguration
	- Microsoft.HybridConnectivity
	- Microsoft.AzureStackHCI
	- Microsoft.Kubernetes
	- Microsoft.KubernetesConfiguration
	- Microsoft.ExtendedLocation
	- Microsoft.ResourceConnector
	- HybridContainerService
	- Microsoft.Attestation
- サブスクリプションに対し、Azure 側の作業をするアカウントに以下の管理権限を付与
	- Azure Stack HCI Administrator
  	- Reader
- Azure Local に関連するオブジェクトを登録するリソースグループを新規作成
	- (リソースグループに対して各オブジェクトが作成される)
- リソースグループに対して、Azure 側で作業をするアカウントに以下の管理権限を付与
	- Azure Connected Machine Onboarding
	- Azure Connected Machine Resource Administrator
	- Key Vault Data Access Administrator
	- Key Vault Secrets Officer　　　      (日本語ポータル作業時は ”キーコンテナーシークレット責任者” を探す)
	- Key Vault Contributor
	- Storage Account Contributor
</details>

## 6. Azure Local 各ノードを Azure Arc で Azure と接続
<details>
	
### 各 Azure Local ノードを Azure Arc に登録するための手順

#### 1. Azure Local ノードと直接接続可能なマシンに接続
#### 2. ノードの管理権限を持つ管理者ユーザーでログオン
#### 3. Configurator application をダウンロードし、起動
- https://aka.ms/ConfiguratorAppForHCI
#### 4. Configurator application にて１つ目のノードの Azure Arc 接続作業
- 4-4-1: マシン名： ノード１の IP アドレスを入力
- 4-4-2: サインイン： administrator
- 4-4-3: パスワードの入力： Azure Local OS インストール後に設定したパスワードを入力
- 4-4-4: Azure Arc エージェントのセットアップ：
	- [開始]-[次へ]
	- 鉛筆マークをクリックし、[MGMT_VM1]を選択して[次へ]
	- 利用するAzure の[サブスクリプションID][リソースグループ][リージョン][テナントID]を入力し[次へ]
	- [完了] 
	- 画面の表示が切り替わり、6つのステップを表示。しばらくすると 6 番目の Arc 構成で認証が促される
	- デバイスコードをコピーし、https://microsoft.com/devicelogin にアクセスしてコードを貼り付け、
	　 Azure Local 展開の権限を持つ Entra ID ユーザーで認証を完了
- 4-4-5: Arc 構成が成功したのち、Azure Portal にて Azure Local マシンが Azure Arc マシンとして登録されているかを確認
#### 5. Configurator application にて2つ目、3つ目のノードの Azure Arc 接続作業
- 4-4-1～4-4-5 と同じ操作を全てのノードに対して実施

</details>

## 7. Azure Local クラスター展開
<details>
	
### クラスター構築作業
-  Azure ポータルの [Azure Arc] - [Azure Local] 管理画面にて、[すべてのシステム (プレビュ―)] を選択
	-  プレビューではない画面にしたい場合は、画面内の [以前のエクスペリエンスに切り替える] をクリックすると GA 済みの画面が表示される
#### 1. [+作成] メニューから [Azure Local インスタンス] を選択
#### 2. 基本タブ
- 2-1: 展開に利用する [サブスクリプション] と [リソースグループ] を選択
	- リソースグループが違うと画面一番下に Arc に登録したサーバー一覧が表示されないので注意
- 2-2: [インスタンス名] には作成するクラスターの名前を入力
- 2-3: [リージョン]はサポートしているリージョンを入力　※ Japan East で OK
- 2-4: [＋マシンの追加] をクリックし、Azure Arc に接続した Azure Local マシン2台を追加
	- 「Arc 拡張機能がありません」と表示されているので、[拡張機能のインストール] をクリック　※ 15分ほどかかる
	-  Azure Portal の Azure Arc 管理画面にて、マシンの一覧にある Azure Local ノードを選択
	- [設定]の下の[拡張機能]をクリックし、4つの拡張モジュールの作成が完了するのを待つ (MDE.Windows は除く)
- 2-5: すべてのノードが準備完了になったら[選択したマシンの確認]をクリック
- 2-6: キーコンテナ―名では [新しいキーコンテナーの作成] をクリックし、右に出てくる画面で [作成] をクリック
	- 繰り返し同じ作業をした場合は既存の Key Vault を削除するか、Key Vault name を変更する事で対応
	- Key Vault は削除しても削除済みリストに残るので、削除済みリストからさらに削除する必要がある
- 2-7: 作業が完了したら [次へ: 構成] をクリック
#### 3. 構成タブ
 - [新しい構成] が選択されていることを確認し [次へ: ネットワーク] をクリック
 	- テンプレートが用意できている場合はテンプレートを利用可能
#### 4.  ネットワークタブ
- ※ ここは実際の環境に合わせて設定をする必要がある
- ※ 以下は NIC4 枚の環境にて、管理＆VM 用ネットワークに MGMT_VM1 と MGMT_VM2 を、ストレージ用に Storage1 と Storage2 を利用する想定
- 4-1: [ストレージのネットワークスイッチ] を選択
- 4-2: [管理とコンピューティングのトラフィックをグループ化する] を選択
- 4-3: インテント名「コンピューティング_管理」に対して [MGMT_VM1] を選択
- 4-4: [+ このトラフィック用の別のアダプターを選択してください] をクリックして [MGMT_VM2] を追加
- 4-5: [ネットワーク設定のカスタマイズ] をクリックして「RDMA プロトコル」を Disabled に変更
- 4-6: インテント名「ストレージ」に対して [Storage1] を選択
- 4-7: 必須項目となっている VLAN ID には[環境に合わせたVLAN ID] を入力
   	- ノード間の通信で利用するためのもの
   	- スイッチレスやスイッチ側ですべての VLAN ID の通信を許可していればデフォルトのままでOK
   	- スイッチ側で設定がされていればその VLAN ID を間違わずに入力すること
- 4-8: [+ このトラフィック用の別のアダプターを選択してください] をクリックして [Storage2] 追加
- 4-9: [環境に合わせた VLAN ID] を入力
- 4-10: [ネットワーク設定のカスタマイズ] をクリックして「RDMA プロトコル」を環境に合わせる　(RoCEv2/iWarp など)
- 4-11: ノードとインスタンスの IP 割り当てが [手動] になっていることを確認　- DHCP 環境があれば自動でもよい
- 4-12: Azure Local が利用する最低 6 つの IP アドレス範囲を用意し、[開始 IP] ~ [終了 IP] 枠に入力
- 4-13: [サブネットマスク　例 255.255.255.0] を入力
- 4-14: [デフォルトゲートウェイの IP アドレス] を入力
- 4-15: [DNS サーバーの IP アドレス] を入力
- 4-16: [サブネットの検証] をクリック
- 4-17: [次へ: 管理] をクリック
#### 5. 管理タブ
- 5-1: Azure から Azure Local クラスターに指示を出す際に利用するロケーション名として [任意のカスタムの場所の名前] を入力
   	- 良く使うので、プロジェクト名や場所、フロアなどを使って、わかりやすい名前を付けておくこと
	- 思い浮かばない時はクラスター名に-cl とつけておくとわかりやすいかも
- 5-2: Azure ストレージアカウント名では、Cloud witness 用に [新規作成]をクリック、さらに右に出てきた内容を確認
	- [作成] をクリックし、Azure ストレージアカウントを作成
- 5-3: ドメイン [例 contoso.com] を入力
- 5-4: OU  [例 OU=test,DC=contoso,DC=com] を入力　　　※4 の Active Directory の準備の際に設定した OU
- 5-5: デプロイアカウントユーザー名を入力　　※ Active Directory の準備の際に指定した Deployment 用のユーザー名
- 5-6: デプロイアカウントユーザーのパスワードを間違えないように入力　※ Deployment 用ユーザーのパスワード
- 5-7: Azure Local マシンのローカル管理者のユーザー名 [administrator] を入力　　※特別な設定をしていなければ administrator で OK
- 5-8: Azure Local マシンのローカル管理者パスワードを間違えないように入力　　※ Azure Local OS インストール後に設定したパスワードを入力
- 5-9: [次へ: セキュリティ] をクリック
#### 6. セキュリティタブ
- [推奨セキュリティ設定] が選択されていることを確認し [次へ: 詳細設定] をクリック
 	- 推奨設定の機能を変更したい場合は [カスタマイズされたセキュリティ設定] をクリックして有効にしたい項目のみを選択
#### 7. 詳細設定タブ
- [ワークロード ボリュームと必要なインフラストラクチャ ボリュームを作成する] が選択されていることを確認し[次へ: タグ] をクリック
	- 既定では、Software Defined Storage プールに ClusterPerformanceHistory/Infrastructure_1 ボリュームと、Azure Local 各ノードを Owner とする論理ボリュームを自動作成してくれる
	- 各ボリュームは既定でシンプロビジョニングされるので、ストレージ容量の効率的な利用はできるものの、ストレージボリュームがいっぱいにならないよう実データ容量のコントロールは必要
	- 固定長ボリュームの作成が必須の場合など、既定と違う構成が必要な場合は[必要なインフラストラクチャ ボリュームのみを作成する]を選択肢、ボリュームを別途作成すること
#### 8. Azure 上のオブジェクトを管理しやすくする任意のタグをつけ、[次へ: 検証] をクリック
- 検証タブが開き、リソース作成ステップ 7 項目が自動実行される
#### 9. 検証タブ
- 9-1: リソース作成用検証ステップの全てが成功になることを確認
- 9-2: [検証を開始] をクリック
- 9-3: 更に 12 個のチェックが行われ、検証が完了したら [次へ: 確認および作成] をクリック
#### 10. 確認および作成タブ
- [作成] をクリックすると Azure Local クラスターの展開が開始される
   - 画面がリソースグループのデプロイ管理画面に遷移するのでしばらくそのままに
   - 画面の表示が変わらなければ、デプロイ管理画面で [更新] をクリックすることで最新の状況を確認できる
   - 手元の 2 ノードで 2 時間半程度かかった


</details>


<!--
__[Azure Local 23H2 展開後の作業はこちら](/toCreateVMs.md)__
-->
