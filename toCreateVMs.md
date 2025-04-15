# Azure Local 23H2 クラスター展開後に必要な作業をまとめました
- __Azure Local 23H2 は Azure ポータルや Azure API から仮想マシンの作成が可能になりました__
- __ただし、Azure ポータルや API から仮想マシンを作成するためには、オンプレミス側の状況を Azure 側にも情報として持っておく必要があります__
- __以下の設定は、そのためのものです__

### Azure Local のネットワークを Azure 側で認識するための仮想ネットワークの定義
- (必要に応じて) Azure ポータルを日本語に変更
- Azure ポータルの [Azure Arc] - [Azure Local] 管理画面にて、[すべてのクラスター (プレビュー)] を選択
- 管理したいクラスター名をクリックすることで Azure Local クラスターの管理画面に遷移
- 左ペインの [論理ネットワーク] をクリックし、[+ 論理ネットワークの作成] をクリック
- [サブスクリプション] と [リソースグループ] を選択
- 任意の [論理ネットワーク名] を入力
- [仮想スイッチ名] を入力
	- Azure Local 構築時に Hyper-V に仮想スイッチが自動作成されているため、それを探して入力
 	- ネットワークの構成次第だが、本環境では ConvergedSwitch(compute_management) となっている
- [次へ: ネットワーク構成] をクリック
　- 仮想マシンの仮想 NIC に自動設定するためのネットワーク情報を入力
	- DHCP の場合は、仮想マシンのネットワークがVLANを利用している場合のみ VLAN ID を入力
 	- 静的の場合は、仮想マシンに割り当てるIPアドレスの範囲、デフォルトゲートウェイや DNS サーバー、VLAN ID などを入力
  		- DHCP サーバーがなくても仮想マシン作成中にIPアドレスを自動(もしくは管理者により手動)で割り当てることが出来る
- [次へ: タグ] をクリック
- リソースグループが違うと画面一番下に Arc に登録したサーバー一覧が表示されないので注意
- 2-2: [Cluster name] を入力
- 2-3: Region はサポートしているリージョンを入力　※ Japan East で OK
- 2-4: Key vault name では [Create a new key vault] をクリックし、右に出てくる画面で [Create] をクリック
	- 繰り返し同じ作業をした場合は既存の Key Vault を削除するか、Key Vault name を変更する事で対応
	- Key Vault は削除しても削除済みリストに残るので、削除済みリストからさらに削除する必要がある
 - 2-5: リソースグループ内の Azuer Stack HCI ノードの一覧が表示されるので、クラスターに参加させるノードにチェックを入れし、[Validate selected servers] をクリック

- 
-  PREVIEW ではない画面にしたい場合は、画面内の [Old Experience] をクリックすると GA 済みの画面が表示される
#### 1. [+Create] メニューから [Azure Local Cluster] を選択
- 
- 
- サブスクリプションに対し、Azure 側の作業をするアカウントに以下の管理権限を付与
	- Azure Local Administrator
 	- Cloud Application Administrator
  	- Reader

