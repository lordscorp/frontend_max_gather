<template>
	<div class="container">
		<h1>Otimizador de Coleta</h1>
		<div style="height: 40px">
			<transition name="slide-fade">
				<div ref="messageBox" v-if="alertMessage.text" :class="'alert alert-'+alertMessage.type" role="alert">
					<span>{{ alertMessage.text }}</span>
					<button type="button" class="close" data-dismiss="alert" v-on:click="hideMessage">&times;</button>
				</div>
			</transition>
		</div>
		<transition name="fade" mode="out-in">
			<form @submit.prevent="validateInputs" v-if="!maxApples">
				<p class="my-4">Preencha os dados para calcular a melhor coleta no intervalo</p>
				<div>
					<!-- Campos para os tamanhos dos intervalos K e L -->
					<div class="row mb-6">
						<label for="marcelo" class="col-sm-4 col-form-label">Quantidade de árvores das quais Marcelo coletará <mark>(K)</mark>:</label>
						<div class="col-sm-2">
							<input id="marcelo" v-model.number="marcelo" type="number" min="0" class="form-control" required>
						</div>
						<label for="carla" class="col-sm-4 col-form-label">Quantidade de árvores das quais Carla coletará <mark>(L)</mark>:</label>
						<div class="col-sm-2">
							<input id="carla" v-model.number="carla" type="number" min="0" class="form-control" required>
						</div>				
					</div>

					<!-- Checkbox que permite utilizar a entrada dos dados do vetor de árvores em lote -->
					<div class="form-check form-check-inline mt-4">
						<input id="batch-insert" type="checkbox" class="form-check-input" v-model="batch" v-on:click="mutateTreeArray">
						<label for="batch-insert" class="form-check-label">Utilizar o modo de inserção em lote</label>
					</div>

					<transition name="fade" mode="out-in">
						<!-- Campo único para inserção do intervalo -->
						<div id="batch-insertion" v-if="batch" class="my-4" key="batchInsertion">
							<label>Vetor de árvores <mark>(A)</mark></label>
							<input type="text" class="form-control" v-model="trees" placeholder="Ex.: [5, 1, 4, 9, 2]">
						</div>

						<!-- Campos dinâmicos acrescentados conforme necessidade -->
						<div id="dynamic-insertion" v-else class="my-4" key="dynamicInsertion">
							<p>Quantidade de maçãs por árvore</p>
							<div class="row my-2" v-for="(index, tree) in trees" v-bind:key="tree">
								<label :for="'tree-' + tree" class="col-sm-4 col-form-label">Árvore {{ tree + 1 }}:</label>
								<div class="col-sm-2">
									<input :id="'tree-' + tree" :ref="'t' + tree" type="number" v-model.number="trees[tree]" class="form-control" v-focus>
								</div>
							</div>
							<div class="d-grid gap-2 col-6 mx-auto">
								<button type="button" class="btn btn-success mx-1 w-25" v-on:click="trees.push(null)" title="Adicionar árvore"><strong>+</strong></button>
								<button type="button" class="btn btn-danger mx-1 w-25" v-on:click="removeLastTree" title="Remover última árvore"><strong>-</strong></button>
							</div>
						</div>
					</transition>
				</div>
				<button type="submit" class="btn btn-primary">Calcular melhor coleta</button>
			</form>
			<!-- Tela de exibição do resultado -->
			<div v-if="maxApples">
				<div id="orchard" class="mw-100">
					<div :class="'tree '+gatherMark(index)" v-for="(apples, index) in trees" v-bind:key="'t'+index">
						<!-- Macieira -->
						<div class="crown"><span>{{apples}}</span></div>
						<div class="trunk">Árvore {{index+1}}</div>
						<div><center><span>{{capitalize(gatherMark(index))}}</span></center></div>
					</div>
				</div>
				<p>Marcelo: coletará maçãs de {{marcelo}} árvores, começando pela árvore {{marceloStart+1}}.</p>
				<p>Carla: coletará maçãs de {{carla}} árvores, começando pela árvore {{carlaStart+1}}.</p>
				<p>Ao todo, ambos coletarão: <strong>{{maxApples}}</strong> maçãs!</p>
				<button v-on:click="maxApples = null" class="btn btn-primary">Realizar novo cálculo</button>
			</div>
		</transition>
	</div>
</template>

<script>
	import axios from 'axios'
	// Componente principal
	// @group Otimizador de Coleta
	export default {
		name: 'Colheita',
		data() {
			return {
				alertMessage: {
					text: null,
					type: 'info'
				},
				gathering: {},
				batch: false,
				trees: ["1", "1"],
				marcelo: null,
				carla: null,
				maxApples: null,
				marceloStart: null,
				carlaStart: null
			}
		},
		mounted: function () {			
			document.querySelector('#marcelo').focus()
		},
		computed: {
			validK () {
				return parseInt(this.marcelo) >= 0
			},
			validL () {
				return parseInt(this.carla) >= 0
			},
			validTreeArray () {
				this.mutateTreeArray()
				return Array.isArray(this.trees) && this.trees.length > 0
			},
			validForm () {
				return this.validK && this.validL && this.validTreeArray
			}
		},
		directives: {
			focus: {
				/**
				 *	Foca no último elemento inserido (input do array de árvores)
				 */
				inserted: function (el) {
					el.focus()
				}
			}
		},
		methods: {
			// @vuese
			/**
			 * Obtém número máximo de maçãs em intervalos contínuos K e L,
			 * dentro de um determinado array de árvores
			 * 
			 * @arg {array} A : Array de inteiros - número de maçãs por árvore
			 * @arg {int} K : Tamanho do intervalo de coleta de Marcelo 
			 * @arg {int} L : Tamanho do intervalo de coleta de Carla
			 */
			getMaxApples(A = [0], K = 1, L = 1) {
				const path = process.env.VUE_APP_API_URL
				let postBody = new FormData();
				postBody.append('A', A)
				postBody.append('K', K)
				postBody.append('L', L)
				axios.post(path, postBody)
				.then((response) => {
					this.gathering = response.data;
					if(parseInt(this.gathering.data[0]) === -1){
						this.showMessage("O intervalo informado é inválido. (Precisamos plantar mais árvores?)", "danger", 8000)
						return
					}
					this.hideMessage()
					this.maxApples = this.gathering.data[0]
					this.marceloStart = this.gathering.data[1]
					this.carlaStart = this.gathering.data[2]
				})
				.catch((e) => {
					console.error(e)
					this.showMessage("Ocorreu um erro no servidor. Por favor, verifique se a API está sendo executada e está aceitando conexões.", "danger", 10000)
				})
			},
			// @vuese
			/**
			 *	Converte string para array (ao editar inserção em lote e voltar para inserção item-a-item)
			 */
			mutateTreeArray() {
				if (typeof this.trees === "string") {
					try {
						let trees = this.trees.replace(/[[\]']/g, "").split(',').map((val) => parseInt(val) || 0)
						this.trees = trees
					}
					catch (e) {
						console.error(e)
					}
				}
				// Converte itens NaN em 0
				for (var i = this.trees.length - 1; i >= 0; i--) {
					this.trees[i] = parseInt(this.trees[i]) || 0
				}
			},
			// @vuese
			/**
			 *	Verifica comprimento do array 'trees' e remove o último item, se possível
			 */
			removeLastTree() {
				if(this.trees.length > 1)
					this.trees.pop()
				else
					this.showMessage("É necessário haver ao menos 1 (uma) árvore.", "warning", 5000)
			},
			// @vuese
			/**
			 *	Exibe mensagens para o usuário.
			 * @arg {string} message : Mensagem a ser exibida.
			 * @arg {string} msgType (opcional) : Tipo de mensagem (para definir estilo css)
			 */
			showMessage(message, msgType = "info", lifespan = 10000) {
				this.alertMessage.text = message
				this.alertMessage.type = msgType
				window.setTimeout((()=>{ this.alertMessage.text = null }), lifespan);
			},
			// @vuese
			/**
			 *	Oculta mensagem atual.
			 */
			hideMessage() {
				this.alertMessage.text = null
			},
			// @vuese
			/**
			 *	Valida os valores de entrada e chama a função getMaxApples()
			 */
			validateInputs() {
				if (this.validForm) {
					this.showMessage("Calculando melhor performance. Por favor, aguarde...")
					this.getMaxApples(this.trees, this.marcelo, this.carla)
				}
				else {
					if (!this.validK) {
						console.error("Valor K inválido")
					}
				}
			},
			// @vuese
			/**
			 *	Verifica se árvore no índice informado deve ter seus frutos coletados por Marcelo ou Carla
			 * @arg {int} treeIndex : Índice do array trees
			 */
			gatherMark(treeIndex){
				if (treeIndex >= this.marceloStart && treeIndex < (this.marceloStart+this.marcelo))
					return "marcelo"
				else if (treeIndex >= this.carlaStart && treeIndex < (this.carlaStart+this.carla))
					return "carla"
				else
					return ""
			},
			// @vuese
			/**
			 *	Coloca a primeira letra do nome em maiúscula
			 *	@arg {string} text : Texto a ser capitalizado
			 */
			 capitalize(text){
			 	if(typeof text === "string")			 		
			 		return text.charAt(0).toUpperCase() + text.slice(1)
			 }
		}
	}
</script>
<style>
	@import '../assets/styles/colheita.css';
</style>