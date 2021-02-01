<template>
	<div class="container">
		<h1>Otimizador de Coleta</h1>
		<form @submit.prevent="validateInputs" class="mt-5">
			<p class="my-4">Preencha os dados para calcular a melhor coleta no intervalo</p>
			<div>
				<!-- Campos para os tamanhos dos intervalos K e L -->
				<div class="row mb-6">
					<label for="marcelo" class="col-sm-4 col-form-label">Quantidade de árvores das quais Marcelo coletará <mark>(K)</mark>:</label>
					<div class="col-sm-2">
						<input id="marcelo" v-model.number="marcelo" type="number" min="1" class="form-control" required>
					</div>
					<label for="carla" class="col-sm-4 col-form-label">Quantidade de árvores das quais Carla coletará <mark>(L)</mark>:</label>
					<div class="col-sm-2">
						<input id="carla" v-model.number="carla" type="number" min="1" class="form-control" required autofocus>
					</div>				
				</div>

				<!-- Checkbox que permite utilizar a entrada dos dados do vetor de árvores em lote -->
				<div class="form-check form-check-inline mt-4">
					<input id="batch-insert" type="checkbox" class="form-check-input" v-model="batch" v-on:click="mutateTreeArray">
					<label for="batch-insert" class="form-check-label">Utilizar o modo de inserção em lote</label>
				</div>

				<!-- Campo único para inserção do intervalo -->
				<div id="batch-insertion" v-if="batch" class="my-4">
					<label>Vetor de árvores <mark>A</mark></label>
					<input type="text" class="form-control" v-model="trees" placeholder="Ex.: [5, 1, 4, 9, 2]">
				</div>

				<!-- Campos dinâmicos acrescentados conforme necessidade -->
				<div id="dynamic-insertion" v-if="!batch" class="my-4">
					<p>Quantidade de maçãs por árvore</p>
					<div class="row my-2" v-for="(index, tree) in trees" v-bind:key="tree">
						<label :for="'tree-' + tree" class="col-sm-4 col-form-label">Árvore {{ tree + 1 }}:</label>
						<div class="col-sm-2">
							<input :id="'tree-' + tree" type="number" v-model.number="trees[tree]" class="form-control">
						</div>
					</div>
				</div>
			</div>
			<button type="submit" class="btn btn-primary">Calcular melhor coleta</button>
		</form>
		<div v-if="maxApples">
			<p>Max apples: {{ maxApples }}</p>
		</div>
	</div>
</template>

<script>
	import axios from 'axios'

	export default {
		name: 'Colheita',
		data() {
			return {
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
		computed: {
			validK () {
				return parseInt(this.marcelo) >= 0
			},
			validL () {
				return parseInt(this.carla) >= 0
			},
			validTreeArray () {
				this.mutateTreeArray()
				return Array.isArray(this.trees)
			},
			validForm () {
				return this.validK && this.validL
			}
		},
		methods: {
			/**
			 * Obtém número máximo de maçãs em intervalos contínuos K e L,
			 * dentro de um determinado array de árvores
			 * 
			 * @param {array} A: Array de inteiros - número de maçãs por árvore
			 * @param {int} K: Tamanho do intervalo de coleta de Marcelo 
			 * @param {int} L: Tamanho do intervalo de coleta de Carla
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
					this.maxApples = this.gathering.data[0]
					this.marceloStart = this.gathering.data[1]
					this.carlaStart = this.gathering.data[2]
				})
				.catch((e) => {
					console.error(e)
				})
			},
			/**
			 *	Converte string para array (ao editar inserção em lote e voltar para inserção item-a-item)
			 */
			mutateTreeArray() {
				if (typeof this.trees === "string") {
					try {
						let trees = this.trees.replace(/[[\]']/g, "").split(',').map((val) => parseInt(val) >= 0 ? parseInt(val) : 0)
						this.trees = trees
					}
					catch (e) {
						console.error(e)
					}
				}
			},
			/**
			 *	Valida os valores de entrada e chama a função getMaxApples()
			 */
			validateInputs() {
				if (this.validForm) {
					this.getMaxApples(this.trees, this.carla, this.marcelo)
				}
				else {
					console.log(this.trees)
				}
			}
		}
	}
</script>