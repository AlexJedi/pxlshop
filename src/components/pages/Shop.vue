<template lang="pug">
  .content
    h1
      | Welcome to our creative store - we sell uncommon design
      | illustrations that smack you right in the heart.
    hr
    section.products
      .gallery
        .filter
          .select-wrap
            select(v-model='selectedCategory')
              option(v-for='category in categories'
              v-bind:value='category') Category: {{ category }}
            span.select-arrow
          .select-wrap
            select(v-model='selectedSort')
              option(v-for='sort in sorts'
              v-bind:value='sort') Sort by: {{ sort }}
            span.select-arrow
        transition-group.gallery-row(name='gallery-anim' tag='div'
        mode="out-in")
            .gallery-product(v-for='(product, index) in filteredProducts'
            v-bind:key='index'
            v-if='index >= productsOnPage * (activePage - 1)\
            && index < productsOnPage * activePage')
              .product-img
                img(v-bind:src='product.url' v-bind:alt='product.name'
                @click='showProductModal(product)')
                .product-actions(v-bind:class='{"product-actions--active": starBoxHover}')
                  a.cart-link(@click='addToCart(product)'
                  v-bind:class='{"cart-link--active": checkInCart(product)}')
                  .star-box(@mouseenter='starBoxHover = true'
                  @mouseleave='starBoxHover = false')
                    span.star.star-link(v-for='star in 5'
                  @click='rateProduct(product, (6 - star))')
              h2.product-title(@click='showProductModal(product)') {{ product.name }}
              span.product-author {{ product.author }}
              span.product-price ${{ product.price + '.00' }}
      aside.sidebar
        .widget-top
          h3.widget-title Top Rated
          ul.top-list
            li.top-product(v-for='product in topProducts')
              .top-product-info
                a.product-title {{ product.name }}
                .product-stars
                  span.star(v-for='n in 5'
                  v-bind:class='{ "star-full": n <= getStars(product) }')
                span.product-price ${{ product.price + '.00' }}
              .top-product-img
                img.product-img(v-bind:src='product.url'
                v-bind:alt='product.name')
        .widget-cart(v-if='cartShow')
          h3.widget-title Cart Review
          ul.cart-list
            li.cart-product(v-for='(product, key) in productsInCart')
              a.product-thumbnail
                img(v-bind:src='product.url' alt='product.name')
              .product-description
                a.product-title {{ product.name }}
                span.product-price $ {{ product.price }}
              a.product-remove(@click='removeFromCart(key)')
          .cart-subtotal
            span Subtotal
            span $ {{ cartAmount() }}
          .cart-links
            a.cart-view(@click='changePage("Cart")') View Cart
            a.cart-checkout(@click='changePage("Checkout")') Checkout
        .widget-payment
          h3.widget-title Payment Options
          p.payment-details
            | Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed
            | do eiusmod tempor incididunt ut labore et dolore magna aliqua.
          .payment-option
            a.option-img
              img(src='../../assets/icons/paypal.svg')
            a.option-img
              img(src='../../assets/icons/visa.svg')
            a.option-img
              img(src='../../assets/icons/mastercard.svg')
    nav.pagination
      a.nav-prev(v-bind:class='{ "nav-disable": activePage == 1 }'
      @click='changePage(activePage - 1)') Prev Page
      .nav-pages
        a.page-num(@click='changeGalleryPage(n)'
        v-for='n in Math.ceil(filteredProducts.length / productsOnPage)'
        v-bind:class='{ "page-active": n == activePage }') {{ n }}
      a.nav-next(@click='changeGalleryPage(activePage + 1)'
      v-bind:class='{ "nav-disable": activePage ==\
      Math.ceil(filteredProducts.length / productsOnPage) }') Next Page

    product(v-on:close='closeProductModal' v-bind='{productModal, topProducts}'
    v-if='productModal')
</template>

<script>
import router from '@/router';
import Firebase from '../../appconfig/firebase';

import Product from './Product';

export default {
  name: 'shop',
  firebase: {
    products: Firebase.dbProductsRef,
  },
  props: ['user', 'productsInCart'],
  components: {
    Product,
  },
  data() {
    return {
      selectedCategory: 'all',
      selectedSort: 'none',
      categories: ['all', 'illustrations', 'patterns', 'photos'],
      sorts: ['none', 'newest', 'popular'],
      productModal: null,
      filteredProducts: {},
      productsOnPage: 12,
      activePage: 1,
      productsStars: {},
      starBoxHover: false,
    };
  },
  created() {
    this.filteredProducts = this.products;
    this.$firebaseRefs.products.on('value', (snapshot) => {
      snapshot.forEach((productSnap) => {
        const key = productSnap.key;
        const value = productSnap.val();
        if (productSnap.val().rating) {
          this.$set(this.productsStars, key, Object.values(value.rating)
            .reduce((sum, val) => sum + val, 0));
        } else {
          this.$set(this.productsStars, key, 0);
        }
      });
    });
  },
  computed: {
    cartShow() {
      return Object.keys(this.productsInCart).length > 0;
    },
    topProducts() {
      return this.sortByRating(this.products).slice(0, 3);
    },
  },
  methods: {
    changePage(page) {
      router.push({
        name: page,
        params: {
          user: this.user,
          productsInCart: this.productsInCart,
        },
      });
    },
    changeGalleryPage(galleryPage) {
      this.activePage = galleryPage;
    },
    showProductModal(product) {
      this.productModal = product;
    },
    closeProductModal() {
      this.productModal = null;
    },
    sortByRating(products) {
      return this.deepClone(products).sort((prodA, prodB) => {
        if (prodA.rating && prodB.rating) {
          const ratingA = Object.values(prodA.rating).reduce(
            (sum, val) => sum + val, 0);
          const ratingB = Object.values(prodB.rating).reduce(
            (sum, val) => sum + val, 0);
          return ratingA <= ratingB ? 1 : -1;
        }
        return prodB.rating ? 1 : -1;
      });
    },
    rateProduct(product, stars) {
      const updates = {};
      updates[`/products/${product['.key']}/rating/${this.user.uid}`] = stars;
      Firebase.dbRef.update(updates);
    },
    addToCart(product) {
      Firebase.dbUsersRef.child(`${this.user.uid}/cart/${product['.key']}`).set({
        name: product.name,
        price: product.price,
        url: product.url,
      });
    },
    checkInCart(product) {
      if (product) {
        return this.productsInCart[product['.key']] !== undefined;
      }
      return false;
    },
    cartAmount() {
      return Object.values(this.productsInCart)
        .reduce((sum, product) => sum + Number(product.price), 0);
    },
    removeFromCart(key) {
      Firebase.dbUsersRef.child(`${this.user.uid}/cart/${key}`).remove();
      this.$delete(this.productsInCart, key);
    },
    getStars(product) {
      return this.productsStars[product['.key']];
    },
    deepClone(obj) {
      return JSON.parse(JSON.stringify(obj));
    },
  },
  watch: {
    selectedCategory(newCategory) {
      if (newCategory === 'all') {
        this.filteredProducts = this.products;
        return;
      }
      this.filteredProducts = this.deepClone(this.products)
        .filter(prod => prod.type === newCategory);
      this.activePage = 1;
    },
    selectedSort(newSort) {
      switch (newSort) {
        case 'popular':
          this.filteredProducts = this.sortByRating(this.products);
          break;
        case 'newest':
          this.filteredProducts = this.deepClone(this.products).sort((prodA, prodB) => (
            Date.parse(prodA.date) < Date.parse(prodB.date) ? 1 : -1
          ));
          break;
        default:
          this.filteredProducts = this.products;
      }
      this.activePage = 1;
    },
  },
};
</script>

<style lang="scss" scoped>

$color-dark: #252525;
$color-grey: #666;
$color-green: #7BEFB2;
$color-light: #fff;

$border: 1px solid rgba(0, 0, 0, 0.2);

h1, h2, h3 {
  color: $color-dark;
}
h1 {
  font-size: 3.8vw;
}
hr {
  border: 0;
  border-top: 2px solid rgba(0, 0, 0, 0.2);
}
.content {
  padding: 12vh 10vw 0 10vw;
}
.products {
  display: flex;
  padding-bottom: 3rem;
}
.filter {
  padding: 2rem 0;
  width: 100%;
  border: 0;
  display: flex;
  justify-content: space-between;
}
.select-wrap {
  flex-basis: 30%;
  padding: 1rem 0;
  position: relative;
  border-bottom: $border;
  font-size: 1.6rem;
  cursor: pointer;
  .select-arrow {
    display: block;
    background-color: $color-light;
    background: url('../../assets/arrow.svg') no-repeat center center;
    position: absolute;
    width: 2rem;
    height: 2rem;
    top: 1rem;
    right: 0;
    z-index: -10;
  }
}
select {
  width: 100%;
  appearance: none;
  background-color: transparent;
  border: 0;
  text-transform: capitalize;
  font-size: 1.6rem;
  color: $color-grey;
  cursor: pointer;
  outline: none;
}
.gallery {
  flex: 2.5;
}
.gallery-anim-enter-active, .gallery-anim-leave-active {
  transition: all 1s;
}
.gallery-anim-leave-to, .gallery-anim-enter-to {
  opacity: 0;
}
.gallery-row {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
}
.gallery-product {
  flex-direction: column;
  margin-bottom: 1rem;
}
.product-img {
  flex-basis: 50%;
  &:hover .product-actions {
    transform: translateX(0) scale(2);
  }
  img {
    transition: all 1s ease;
    &:hover {
      transform: scale(1.2);
    }
  }
}
.product-actions {
  position: absolute;
  overflow: visible;
  top: calc(50% - 2rem);
  right: 2rem;
  width: 2rem;
  transform: translateX(300%) scale(2);
  transition: 1s;
}

@for $i from 1 through 4 {
  .star-box:hover .star-link:nth-of-type(#{$i+1}) {
    $length: -2rem*$i;
    transform: translateX($length);
  }
}
.cart-link,
.star-link {
  &:hover {
    background-color: $color-green;
  }
}
.cart-link {
  display: block;
  height: 2rem;
  width: 100%;
  background: $color-light url('../../assets/icons/cart.svg') no-repeat center center;
  background-size: 40%;
}
.cart-link--active {
  background-color: $color-green;
  pointer-events: none;
}
.star-box {
  position: relative;
  height: 2rem;
}
.star-link {
  display: block;
  position: absolute;
  top: 0;
  left: 0;
  height: 2rem;
  width: 2rem;
  background-color: $color-light;
  cursor: pointer;
  text-align: center;
  line-height: 2rem;
  vertical-align: middle;
  font-size: 1rem;
  transition: 1s;
  &:hover {
    &::before {
      content: '\2605';
      color: $color-dark;
    }
  }
  &:hover ~ .star-link {
    &::before {
      content: '\2605';
      color: $color-dark;
    }
  }
}
.product-title {
  margin-bottom: 0.3rem;
  cursor: pointer;
}
.product-price, .product-author {
  font-size: 1vw;
}
.product-author {
  padding-bottom: 0.5rem;
}

// sidebar section
.sidebar {
  flex: 1;
  display: flex;
  flex-direction: column;
  margin-left: 3vw;
}
.widget-title {
  letter-spacing: 0.1rem;
  padding: 2rem 0;
  margin-bottom: 0;
}
.top-list {
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
}
.top-product {
  display: flex;
  border-bottom: $border;
  padding: 2rem 0;
}
.top-product-info {
  flex: 3;
  display: flex;
  flex-direction: column;
  .product-stars, .product-price {
    flex: 1;
    padding-top: 1rem;
  }
}
.star {
  &::before {
    content: '\2606';
    color: $color-grey;
  }
}
.star-full {
  &::before {
    content: '\2605';
    color: $color-green;
  }
}
.top-product-img {
  flex-basis: 10rem;
}
.widget-payment {
  flex-basis: 15rem;
  padding: 0 2rem 2.5rem 2rem;
  background-color: darken($color-light, 5);
  margin-top: 2rem;
}
.payment-details {
  margin-top: 0;
  text-align: justify;
}
.payment-option {
  display: flex;
  justify-content: space-around;
}
.option-img {
  flex-basis: 20%;
  &:hover {
    opacity: 0.8;
  }
}
.widget-cart {
  flex-basis: 15rem;
  margin-top: 2rem;
  margin-bottom: 2rem;
}
.pagination {
  display: flex;
  width: 100%;
  padding: 5rem 0;
  justify-content: space-between;
  border-top: 2px solid rgba(0, 0, 0, 0.2);
  >a {
    flex-basis: 20%;
    position: relative;
    opacity: 0.7;
    &::before {
      content: '';
      position: absolute;
      background: url('../../assets/arrow.svg') no-repeat top center;
      width: 2rem;
      height: 2rem;
      top: -10%;
    }
    &:hover {
      opacity: 1;
      color: $color-grey;
    }
  }
}
.nav-prev {
  text-align: right;
  &::before {
    left: 0;
    transform: rotate(90deg);
  }
}
.nav-next {
  &::before {
    right: 0;
    transform: rotate(-90deg);
  }
}
.nav-disable {
  opacity: 0.4;
  &:hover {
    color: inherit;
  }
  pointer-events: none;
}
.nav-pages {
  flex-basis: 10%;
  display: flex;
  justify-content: space-around;
}
.page-num {
  color: $color-dark;
  &:hover {
    color: $color-green;
  }
}
.page-active {
  color: $color-green;
  pointer-events: none;
}

@media screen and (max-width: 991px) {
  .gallery-product {
    flex-basis: calc(90% / 2);
  }
  .product-price, .product-author {
    font-size: 1.5vw;
  }
  .select-wrap {
    flex-basis: 40%;
  }
}

@media screen and (max-width: 800px) {
  h1 {
    font-size: 5vw;
  }
  .filter {
    flex-direction: column;
  }
  .content {
    padding: 12vh 15vw 0 15vw;
  }
  .products {
    flex-direction: column;
  }
  .gallery, .sidebar {
    flex: 1;
  }
  .gallery-product {
    flex-basis: 100%;
  }
  .product-title {
    font-size: 3vw;
  }
  .product-price, .product-author {
    font-size: 3vw;
  }
  .cart-link {
    height: 3rem;
  }
}
</style>
