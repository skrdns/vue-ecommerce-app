<template>
  <div
    class="shop-container p-10 bg-gray-900 bg-opacity-90 rounded-lg mt-20 text-white text-center w-11/12 max-w-7xl mx-auto"
  >
    <h1 class="text-4xl font-extrabold mb-8">SkeletonStore</h1>

    <div class="filter-container mb-10 p-5 bg-purple-800 bg-opacity-80 rounded-lg shadow-lg">
      <div class="filter-item mb-4">
        <label for="search" class="text-lg font-semibold">Пошук за назвою:</label>
        <input
          type="text"
          id="search"
          v-model="filters.search"
          @input="updateURL"
          placeholder="Введіть назву..."
          class="w-full p-3 rounded-md text-black focus:outline-none focus:ring-2 focus:ring-purple-600"
        />
      </div>

      <div class="filter-item mb-4 flex gap-5">
        <div class="flex-1">
          <label for="min-price" class="text-lg font-semibold">Ціна від:</label>
          <input
            type="number"
            id="min-price"
            v-model="filters.minPrice"
            @input="updateURL"
            placeholder="Мінімальна ціна"
            class="w-full p-3 rounded-md text-black focus:outline-none focus:ring-2 focus:ring-purple-600"
          />
        </div>
        <div class="flex-1">
          <label for="max-price" class="text-lg font-semibold">Ціна до:</label>
          <input
            type="number"
            id="max-price"
            v-model="filters.maxPrice"
            @input="updateURL"
            placeholder="Максимальна ціна"
            class="w-full p-3 rounded-md text-black focus:outline-none focus:ring-2 focus:ring-purple-600"
          />
        </div>
      </div>

      <div class="filter-item mb-4">
        <label for="sort-by" class="text-lg font-semibold">Сортувати за:</label>
        <select
          v-model="sortBy"
          @change="updateSorting"
          id="sort-by"
          class="w-full p-3 rounded-md text-black focus:outline-none focus:ring-2 focus:ring-purple-600"
        >
          <option value="default">За замовчуванням</option>
          <option value="priceLowHigh">Ціна: від найнижчої до найвищої</option>
          <option value="priceHighLow">Ціна: від найвищої до найнижчої</option>
          <option value="titleAZ">Назва: від А до Я</option>
          <option value="titleZA">Назва: від Я до А</option>
        </select>
      </div>

      <div class="filter-item mt-6">
        <button
          @click="resetFilters"
          class="w-full bg-white text-black p-3 rounded-md font-semibold hover:bg-gray-200 transition-colors"
        >
          Скинути фільтри
        </button>
      </div>
    </div>

    <div v-if="filteredProducts.length > 0" class="product-list grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
      <div
        class="product-card flex flex-col items-center bg-purple-800 p-5 rounded-lg shadow-lg hover:shadow-2xl transform hover:translate-y-1 transition-all"
        v-for="product in paginatedProducts"
        :key="product.id"
      >
        <img
          v-if="product.image"
          :src="product.image"
          :alt="product.title"
          class="product-image w-full h-64 object-cover rounded-md mb-4"
        />
        <h3 class="text-lg font-semibold text-white mb-2">{{ product.title }}</h3>
        <p class="text-sm text-gray-300 mb-4">{{ product.description }}</p>
        <p class="text-lg font-semibold text-white mb-4">{{ product.price }} $</p>

        <div class="button-group flex gap-4">
          <button
            @click="addToCart(product)"
            class="add-to-cart-button bg-green-500 hover:bg-green-600 text-white px-4 py-2 rounded-md font-bold transition-colors"
          >
            Додати до кошика
          </button>
          <button
            @click="toggleFavorite(product)"
            :class="isFavorite(product.id) ? 'bg-red-500 hover:bg-red-600' : 'bg-gray-500 hover:bg-gray-600'"
            class="add-to-favorites-button text-white px-4 py-2 rounded-md font-bold transition-colors"
          >
            {{ isFavorite(product.id) ? 'Улюблене' : 'Додати в улюблене' }}
          </button>
        </div>
      </div>
    </div>

    <div v-else class="no-products text-xl font-bold text-gray-300">
      Немає товарів, що відповідають фільтрам.
    </div>

    <div class="pagination mt-6" v-if="filteredProducts.length > 0">
      <button
        @click="goToPage(currentPage - 1)"
        :disabled="currentPage === 1"
        class="prev-page bg-purple-600 text-white p-2 rounded-md hover:bg-purple-700"
      >
        Попередня
      </button>
      <span class="page-number text-white mx-4">{{ currentPage }} / {{ totalPages }}</span>
      <button
        @click="goToPage(currentPage + 1)"
        :disabled="currentPage === totalPages"
        class="next-page bg-purple-600 text-white p-2 rounded-md hover:bg-purple-700"
      >
        Наступна
      </button>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  name: 'Shop',
  data() {
    return {
      products: [],
      cart: [],
      favorites: JSON.parse(localStorage.getItem('favorites')) || [],
      filters: {
        search: this.$route.query.search || '',
        minPrice: this.$route.query.minPrice || '',
        maxPrice: this.$route.query.maxPrice || '',
      },
      sortBy: 'default',
      currentPage: 1,
      itemsPerPage: 8,
      maxPages: 12,
    };
  },
  computed: {
    sortedProducts() {
      let sorted = [...this.products];
      switch (this.sortBy) {
        case 'priceLowHigh':
          return sorted.sort((a, b) => a.price - b.price);
        case 'priceHighLow':
          return sorted.sort((a, b) => b.price - a.price);
        case 'titleAZ':
          return sorted.sort((a, b) => a.title.localeCompare(b.title));
        case 'titleZA':
          return sorted.sort((a, b) => b.title.localeCompare(a.title));
        default:
          return sorted;
      }
    },
    filteredProducts() {
      return this.sortedProducts.filter((product) => {
        const hasValidTitle = product.title && product.title !== 'New Product';
        const matchesSearch = product.title.toLowerCase().includes(this.filters.search.toLowerCase());
        const matchesPrice =
          (!this.filters.minPrice || product.price >= this.filters.minPrice) &&
          (!this.filters.maxPrice || product.price <= this.filters.maxPrice);
        const hasValidImage = product.image;
        return hasValidTitle && hasValidImage && matchesSearch && matchesPrice;
      });
    },
    totalPages() {
      const pages = Math.ceil(this.filteredProducts.length / this.itemsPerPage);
      return pages > this.maxPages ? this.maxPages : pages;
    },
    paginatedProducts() {
      const start = (this.currentPage - 1) * this.itemsPerPage;
      const end = start + this.itemsPerPage;
      return this.filteredProducts.slice(start, end);
    },
  },
  methods: {
    async fetchProducts() {
      try {
        const { data } = await axios.get('https://api.escuelajs.co/api/v1/products');
        const validateImage = async (url) => {
          try {
            const response = await fetch(url, { method: 'HEAD' });
            return response.ok;
          } catch {
            return false;
          }
        };
        const productsWithValidImages = [];
        for (const product of data) {
          const isValidImage = product.images[0] && (await validateImage(product.images[0]));
          if (isValidImage) {
            productsWithValidImages.push({
              id: product.id,
              title: product.title,
              price: product.price,
              image: product.images[0],
              description: product.description
                ? product.description.slice(0, 100) + '...'
                : 'Нет описания',
            });
          }
        }
        this.products = productsWithValidImages;
      } catch (err) {
        console.error('Ошибка загрузки продуктов:', err);
      }
    },
    addToCart(product) {
      const existingProduct = this.cart.find((item) => item.id === product.id);
      if (existingProduct) {
        existingProduct.quantity += 1;
      } else {
        this.cart.push({ ...product, quantity: 1 });
      }
      localStorage.setItem('cart', JSON.stringify(this.cart));
    },
    toggleFavorite(product) {
      const index = this.favorites.findIndex((item) => item.id === product.id);
      if (index > -1) {
        this.favorites.splice(index, 1);
      } else {
        this.favorites.push(product);
      }
      localStorage.setItem('favorites', JSON.stringify(this.favorites));
    },
    isFavorite(productId) {
      return this.favorites.some((product) => product.id === productId);
    },
    resetFilters() {
      this.filters = { search: '', minPrice: '', maxPrice: '' };
      this.updateURL();
    },
    updateURL() {
      this.$router.replace({ query: { ...this.filters } });
    },
    updateSorting() {
      this.currentPage = 1;
    },
    goToPage(page) {
      if (page < 1 || page > this.totalPages) return;
      this.currentPage = page;
    },
  },
  async created() {
    await this.fetchProducts();
  },
};
</script>


<style scoped>
.product-image {
  width: 100%;
  height: 16rem;
  object-fit: cover;
  border-radius: 0.5rem;
}
.no-products {
  margin: 50px 0;
}
.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 20px;
}
</style>
