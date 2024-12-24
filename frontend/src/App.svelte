<script>
	import { onMount } from 'svelte';
	import axios from 'axios';
	
	const API_URL = 'http://localhost:3000';
	
	let rooms = [];
	let selectedRoom = null;
	let numberOfPeople = 1;
	let message = '';
	let errorMessage = '';
	let loading = false;
	let additionalPayments = 1; // Nueva variable para manejar pagos adicionales
	let showPaymentButton = true; 
	
	const fetchRooms = async () => {
	  loading = true;
	  try {
		const response = await axios.get(`${API_URL}/rooms`);
		rooms = response.data;
		errorMessage = '';
	  } catch (error) {
		errorMessage = 'Error al cargar las habitaciones';
		console.error(error);
	  } finally {
		loading = false;
	  }
	};
	
	onMount(fetchRooms);
	
	const selectRoom = (room) => {
	  selectedRoom = room;
	  numberOfPeople = 1;
	  message = '';
	  errorMessage = '';
	};
	
	const reserveRoom = async () => {
	  if (!selectedRoom || numberOfPeople < 1 || numberOfPeople > selectedRoom.capacity * 2) { // Permitir el doble de capacidad
		errorMessage = 'Por favor, seleccione un número de personas válido (no mayor al doble de la capacidad)';
		return;
	  }
  
	  loading = true;
	  try {
		await axios.post(`${API_URL}/rooms/${selectedRoom._id}/reserve`, {
		  people: numberOfPeople
		});
		await fetchRooms();
		selectedRoom = rooms.find(r => r._id === selectedRoom._id);
		message = 'Habitación reservada exitosamente';
		errorMessage = '';
	  } catch (error) {
		errorMessage = error.response?.data?.message || 'Error al reservar';
		console.error(error);
	  } finally {
		loading = false;
	  }
	};
	
	const payForRoom = async () => {
		if (!selectedRoom) return;

		loading = true;
		try {
			// Realizar el pago por el monto completo multiplicado por los pagos adicionales
			await axios.post(`${API_URL}/rooms/${selectedRoom._id}/pay`, {
			amount: selectedRoom.price * additionalPayments // Calculando el monto total
			});

			// Opcionalmente, puedes refrescar las habitaciones para que muestren los cambios
			await fetchRooms();
			selectedRoom = rooms.find(r => r._id === selectedRoom._id);
			message = 'Pago procesado exitosamente';
			errorMessage = '';
			
			// Iniciar el temporizador para ocultar el botón después de 3 segundos
		} catch (error) {
			errorMessage = error.response?.data?.message || 'Error al procesar el pago';
			console.error(error);
		} finally {
			loading = false;
		}
		};
  </script>
  
  <main class="container">
	<h1 class="title">Sistema de Reservas</h1>
  
	{#if loading}
	  <div class="loading">Cargando...</div>
	{/if}
  
	{#if errorMessage}
	  <div class="error">{errorMessage}</div>
	{/if}
  
	{#if message}
	  <div class="success">{message}</div>
	{/if}
  
	<div class="rooms-grid">
	  {#each rooms as room}
		<div 
		  class="room-card {selectedRoom?._id === room._id ? 'selected' : ''}"
		  class:unavailable={room.status === 'reserved'}
		  on:click={() => selectRoom(room)}
		>
		  <h3>Habitación {room.number}</h3>
		  <p class="room-type">{room.type}</p>
		  <p>{room.description}</p>
		  <p class="price">${room.price}</p>
		  <p>Capacidad: {room.capacity} personas</p>
		  <p class="status">Estado: {room.status === 'free' ? 'Disponible' : 'Reservada'}</p>
		</div>
	  {/each}
	</div>
  
	{#if selectedRoom}
	  <div class="reservation-panel">
		<h2>Gestionar Habitación {selectedRoom.number}</h2>
		
		{#if selectedRoom.status === 'free'}
		  <div class="reservation-form">
			<label>
			  Número de personas:
			  <input 
				type="number" 
				bind:value={numberOfPeople} 
				min="1" 
			  />
			</label>
			<button 
			  on:click={reserveRoom}
			  disabled={loading || numberOfPeople > selectedRoom.capacity }
			>
			  Reservar
			</button>
		  </div>
		{:else if selectedRoom.status === 'reserved'}
		  <div class="payment-form">
			<p>Monto a pagar: ${selectedRoom.price}</p>
				<button 
					on:click={payForRoom}
					disabled={additionalPayments < 1}
				>
					Procesar Pago
				</button>
		  </div>
		{/if}
	  </div>
	{/if}
  </main>
  
  <style>
	.container {
	  max-width: 1200px;
	  margin: 0 auto;
	  padding: 20px;
	}
  
	.title {
	  text-align: center;
	  margin-bottom: 30px;
	}
  
	.loading {
	  text-align: center;
	  padding: 20px;
	  color: #666;
	}
  
	.error {
	  background-color: #fee;
	  color: #c00;
	  padding: 10px;
	  border-radius: 4px;
	  margin-bottom: 20px;
	}
  
	.success {
	  background-color: #efe;
	  color: #0c0;
	  padding: 10px;
	  border-radius: 4px;
	  margin-bottom: 20px;
	}
  
	.rooms-grid {
	  display: grid;
	  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
	  gap: 20px;
	  margin-bottom: 30px;
	}
  
	.room-card {
	  padding: 20px;
	  border: 2px solid #ddd;
	  border-radius: 8px;
	  cursor: pointer;
	  transition: all 0.3s ease;
	}
  
	.room-card:hover {
	  transform: translateY(-2px);
	  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
	}
  
	.room-card.selected {
	  border-color: #4CAF50;
	}
  
	.room-card.unavailable {
	  opacity: 0.7;
	  background-color: #f5f5f5;
	}
  
	.room-type {
	  color: #666;
	  font-style: italic;
	}
  
	.price {
	  font-size: 1.2em;
	  font-weight: bold;
	  color: #4CAF50;
	}
  
	.status {
	  margin-top: 10px;
	  font-weight: bold;
	}
  
	.reservation-panel {
	  margin-top: 30px;
	  padding: 20px;
	  border-top: 1px solid #ddd;
	}
  
	.reservation-form, .payment-form {
	  display: flex;
	  flex-direction: column;
	  gap: 15px;
	  max-width: 300px;
	  margin: 0 auto;
	}
  
	input {
	  padding: 8px;
	  border: 1px solid #ddd;
	  border-radius: 4px;
	  margin-top: 5px;
	}
  
	button {
	  padding: 12px 24px;
	  background-color: #4CAF50;
	  color: white;
	  border: none;
	  border-radius: 4px;
	  cursor: pointer;
	  transition: background-color 0.3s;
	}
  
	button:hover:not(:disabled) {
	  background-color: #45a049;
	}
  
	button:disabled {
	  background-color: #cccccc;
	  cursor: not-allowed;
	}
  </style>
  