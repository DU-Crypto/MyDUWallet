<template>

      <section class="back-cover">
    <div class="home homepage-home">
      <vue-header></vue-header>
        <div class="row justify-content-center">
            <div class="col-md-11">
                <div class="card card-default">
                    <transition name="fade">
                    <div v-if="!loggedin" class="card-body">
                      <div class="form-group">
                        <label for="json-file">JSON File</label>
                        <input  type="file" id="json-file"/>
                        <input class="form-control" placeholder="Phrase" v-model="phrase" type="password">
                      </div>
                        <div class="form-group">
                          <div class="form-check">
                            <label class="form-check-label">
                              <input type="radio" class="form-check-input" value="true" selected v-model="testnet">TestNet
                            </label>
                          </div>
                          <div class="form-check">
                            <label class="form-check-label">
                              <input type="radio" class="form-check-input" value="false" disabled v-model="testnet">MainNet
                            </label>
                          </div>
                          <button v-on:click="createAccount()" class="btn btn-primary btn-gap btn-custom">Login With Private Key</button>
                          <button v-on:click="generateAccount()" class="btn btn-primary btn-gap btn-custom">Create Wallet</button>

                        </div>


                    </div>
                    <div v-else class="card-body">
                      <div class="row">
                        <div class="col-md-1"><span class="small-logo"><img src="static/images/logo-small.png"></span></div>
                        <div class="col-md-4 balance-section">
                        <h2>Account Balance:</h2>

                        </div>
                        <div class="col-md-7">
                            <div class="address-title">Address</div>
                            <div class="address-section">{{balance.address}}</div>
                        </div>
                    </div>


                        <input placeholder="NEP-5 Script Hash" class="form-control" v-model="scriptHash">
                        <button class="btn btn-primary btn-gap btn-custom" v-on:click="addToken(scriptHash)">Add Token</button>
                        <button class="btn btn-primary btn-gap btn-custom" v-on:click="claimGas">Claim Gas</button>
                        <button class="btn btn-primary btn-gap btn-custom" v-on:click="sendAssetPrompt">Send <i class="icon" data-icon="d"></i></button>
                        <button class="btn btn-primary btn-gap btn-custom" data-toggle="modal" data-target="#transactionModal">Get Transactions</button>
                        <div v-if="balance.balance != null" v-for="b in balance.balance">
                          <span  class="luv-count">{{b.amount}} - {{b.asset}}</span>
                          <br>
                        </div>
                        </div>
                  </transition>
                </div>
            </div>
        </div>
        <!-- Transactions -->
        <!-- Modal -->
          <div id="transactionModal" class="modal fade" role="dialog">
            <div class="modal-dialog modal-lg">

              <!-- Modal content-->
              <div class="modal-content">
                <div class="modal-header">
                  <h4 class="modal-title">My Transactions</h4>
                </div>
                <div class="modal-body table-responsive">
                  <table class="table">
                  <thead>
                    <tr>
                      <th>Type</th>
                      <th>Txid</th>
                      <th>Transfers</th>
                      <th>Amount</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="transaction in transactions">
                      <td>{{transaction.type}}</td>
                      <td>{{transaction.txid}}</td>
                      <td>
                        <ul>
                          <li v-for="transfer in transaction.transfers">
                            {{transfer.address_from}} -> {{transfer.address_to}}
                          </li>
                        </ul>
                      </td>
                      <td>
                        <div v-if="transaction.vin.length > 0">
                          <ul>
                            <li v-for="vin in transaction.vin">{{vin.value}}</li>
                          </ul>
                        </div>
                      </td>
                    </tr>

                  </tbody>
                </table>

                </div>
                <div class="modal-footer">
                  <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
                </div>
              </div>

            </div>
          </div>

        <!---- Transactions -->
        <!-- Modal -->
        <div id="assetModal" class="modal fade" role="dialog">
          <div class="modal-dialog">

            <!-- Modal content-->
            <div class="modal-content">
              <div class="modal-header">
                <h4 class="modal-title">Send Asset</h4>
              </div>
              <div class="modal-body">
                <div class="form-group">
                  <label for="sendAddr">Address</label>
                  <input id="sendAddr" v-model="sendAddr" class="form-control">
                </div>
                <div v-if="balance != null" class="form-group">
                  <label for="gas-asset">Assets/Tokens:</label>
                  <select placeholder="Select Asset/Token" id="token-asset"  v-model="tokenSend" class="form-control">
                    <option v-for="balance in balance.balance" :value="balance">{{balance.asset}}</option>
                  </select>
                  <input id="neo-asset" type="number" v-model="neoSend" class="form-control">

                </div>

              </div>
              <div class="modal-footer">
                <button type="button" class="btn btn-primary"  v-on:click="sendAsset">Send <span class="glyphicon glyphicon-send"></span></button>
                <button type="button" class="btn btn-danger" data-dismiss="modal">Close <span class="glyphicon glyphicon-remove"></span></button>
              </div>
            </div>

          </div>
        </div>
        <br>
        <vue-footer></vue-footer>
    </div>
  </section>
</template>

<script>
    export default {
        mounted() {
            console.log('Component mounted.')

        },
        data(){
          return {
            Neon: {},
            account: null,
            db: {},
            wallet: null,
            balance: null,
            privateKey: '',
            phrase: '',
            loggedin:false,
            tokens: [],
            neoSend:0,
            gasSend:0,
            sendAddr:'',
            gasPrice: 0.00,
            neoPrice: 0.00,
            testnet: false,
            transactions:{},
            server: {},
            scriptHash: '',
            tokenSend: ''
          }
        },
        created(){
          this.db = require('db.js');

          this.Neon = Neon.default;

          this.checkLogin()
          if ('serviceWorker' in navigator) {
            window.addEventListener('load', function() {
              navigator.serviceWorker.register('/background.js').then(function(registration) {

              }, function(err) {
                // registration failed :(
                console.log('ServiceWorker registration failed: ', err);
              });
            });
          }

        },
        methods:{
          checkLogin: function(){
            if(sessionStorage.encryptedPrivateKey != null && sessionStorage.phrase != null){
              this.account = new wallet.Account(sessionStorage.encryptedPrivateKey).decrypt(sessionStorage.phrase);
              this.getBalance();
              this.loggedin = true;
              return true
            }
            else{
              return false
            }

          },
          getTransactions: function(){
            var that = this;
            axios.get('https://neoscan.io/api/main_net/v1/get_last_transactions_by_address/'+this.account.address+'/1').then(data => {

              that.transactions = data.data;
            }).catch(err => {
              alert('There was an error getting transactions')
            })
          },
          createAccount:function(){
            var file = document.getElementById('json-file').files[0];
            var reader = new FileReader();
            var that = this;
            reader.onload = (event) => {
              var keyStore = JSON.parse(event.target.result);
              this.account = new wallet.Account(keyStore.encryptedPrivateKey).decrypt(this.phrase);
              sessionStorage.encryptedPrivateKey = keyStore.encryptedPrivateKey;
              sessionStorage.phrase = that.phrase;


              var balance = new wallet.Balance({net: 'MainNet', address: this.account.address})

              that.getBalance();


            };

            reader.readAsText(file);



          },
          generateAccount:function(){
            var ans = prompt('Create Phrase!')
            var that = this;
            if (ans != ''){
            sessionStorage.phrase = ans;
            this.account =new wallet.Account(this.Neon.create.privateKey()).encrypt(ans);

              var balance = new wallet.Balance({net: 'MainNet', address: this.account.address})


            this.downloadKeystore();
            var that = this;
            that.getBalance();


          }
          else{
            alert('Must create password for wallet!')
            }

          },
          downloadKeystore: function () {
            var accountDetails = {
              'WIF': this.account.WIF,
              'privateKey': this.account.privateKey,
              'publicKey': this.account.publicKey,
              'scriptHash': this.account.scriptHash,
              'address': this.account.address,
              'encryptedPrivateKey': this.account.encrypted
            }
            sessionStorage.encryptedPrivateKey = accountDetails.encryptedPrivateKey;

            var FileSaver = require('file-saver')
            var blob = new Blob([JSON.stringify(accountDetails)], {type: 'text/plain;charset=utf-8'})
            FileSaver.saveAs(blob, this.account.address+'.json')
          },
          addTokenToDb: function(token){
            var that = this;

            this.server.neoTokens.put(token).then(item => {
              console.log(item);

              that.balance.balance.push(token);

            });

          },
          addToken:function(scriptHash){
            var that = this;
          api.nep5.getToken("http://seed3.cityofzion.io:8080",scriptHash,this.balance.address).then(data => {
            let token = {
              asset : data.symbol,
              amount: data.balance,
              scriptHash: scriptHash
            }
            that.addTokenToDb(token);
          });
          },
          addTokens:function(token){
            var that = this;

            that.balance.balance.push(token);

          },
          sendAssetPrompt: function(){
            console.log('sendAsset Clicked!');
            $("#assetModal").modal()
          },
          sendAsset: function(){
                    // We want to send 1 NEO and 1 GAS to ALq7AWrhAueN6mJNqk6FHJjnsEoPRytLdW
          if(this.tokenSend.asset == 'NEO'){
            var intent = api.makeIntent({NEO:this.neoSend}, this.sendAddr)
            const config = {
              net: 'MainNet', // The network to perform the action, MainNet or TestNet.
              address: this.balance.address,  // This is the address which the assets come from.
              privateKey: this.account.privateKey,
              intents: intent
            }
            var that = this;
            this.Neon.sendAsset(config)
            .then(config => {
              alert("Success txid: "+config.response.txid)

              that.getBalance();
          })

          }
          else if(this.tokenSend.asset == 'GAS'){
          var intent = api.makeIntent({GAS:this.neoSend}, this.sendAddr)
          const config = {
            net: 'MainNet', // The network to perform the action, MainNet or TestNet.
            address: this.balance.address,  // This is the address which the assets come from.
            privateKey: this.account.privateKey,
            intents: intent
          }
          var that = this;
          this.Neon.sendAsset(config)
          .then(config => {
            alert("Success txid: "+config.response.txid)

            that.getBalance();
        })

          }
          else{
            /*
            api.nep5.doTransferToken('MainNet', this.tokenSend.scriptHash, this.account.privateKey, this.sendAddr, this.tokenSend, 0, null).then(data =>{
              console.log(data);
            })
            */
            var that = this;
            var hash = this.tokenSend.scriptHash;
            var sendAddr = this.Neon.get.scriptHashFromPublicKey(this.sendAddr);
            console.log(sendAddr);
            var toAddrScriptHash = this.Neon.u.reverseHex(sendAddr);
            var invoke = {scriptHash:hash, operation: 'transfer', args: [this.Neon.u.reverseHex(this.account.scriptHash), toAddrScriptHash, this.Neon.u.str2hexstring(this.neoSend)] }
            let transaction1 = this.Neon.create.script(invoke)
            alert(transaction1)
          }


      },
      getBalance:function(){
        var that = this;
      /*  api.cmc.getPrice('GAS').then(res => {
          that.gasPrice = res;
        })
        api.cmc.getPrice('NEO').then(res => {
          that.neoPrice = res;
        })
        */
        axios.get('https://neoscan.io/api/main_net/v1/get_balance/'+this.account.address).then(obj => {
          that.balance = obj.data;
          that.balance.balance = that.balance.balance.filter(function(i){

          return i.asset == 'NEO' || i.asset == 'GAS';
          });

          this.getTransactions();
          this.getTokens();
          that.loggedin = true;
      })
      .catch(config => {
        console.log(config)
      })
    },
    getTokens: function(){

      var that = this;
      this.db.open({
          server: 'my-du-wallet',
          version: 1,
          schema: {
              neoTokens: {
                  key: {keyPath: 'id', autoIncrement: true},
                  // Optionally add indexes
                  indexes: {

                      scriptHash: {unique: true}
                  }
              },
              ethTokens: {
                key: {keyPath: 'id', autoIncrement: true},
                indexes:{
                  scriptHash: {unique: true}
                }
              }
          }
      }).then(function (s) {
          that.server = s;
          that.server.neoTokens.query()
            .all()
            .execute()
            .then(function (results) {
                // do something with the results
                results.forEach(function(element) {
                  console.log(element);
                  that.addTokens(element)
                });
            });
      });
    },
          claimGas: function(){

            this.Neon.claimGas({net: 'MainNet',address:this.balance.address,privateKey: this.account.privateKey})
            .then(config => {
              alert('Gas CLAIMED!')
              this.getBalance();
            })
            .catch(config => {
              console.log(config)
            })
          }
        }
    }
</script>
<style lang="scss" scoped>
  @import "NeoWallet.scss";
</style>
