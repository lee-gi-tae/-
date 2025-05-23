<template>
  <div class="product-detail-container">
    <!-- Loading and error states -->
    <div v-if="loading" class="loading-container">
      <div class="loading-spinner"></div>
      <p>상품 정보를 불러오는 중입니다...</p>
    </div>

    <div v-else-if="error" class="error-container">
      <div class="error-icon">⚠️</div>
      <h3>오류가 발생했습니다</h3>
      <p>{{ error }}</p>
      <button @click="loadProductDetails" class="retry-button">다시 시도</button>
    </div>

    <!-- Product details -->
    <template v-else-if="product">
      <!-- Header with navigation and actions -->
      <div class="page-header">
        <button @click="goBack" class="back-button">
          <i class="fas fa-arrow-left"></i>
          <span>상품 목록으로</span>
        </button>

        <div class="header-actions">
          <button @click="sharePage" class="share-button">
            <i class="fas fa-share-alt"></i>
          </button>
          <button
            @click="toggleFavorite"
            :class="['favorite-button', { 'is-favorite': isFavorite }]"
          >
            <i :class="isFavorite ? 'fas fa-heart' : 'far fa-heart'"></i>
            <span>{{ isFavorite ? '즐겨찾기 해제' : '즐겨찾기 추가' }}</span>
          </button>
        </div>
      </div>

      <!-- Product header card -->
      <div class="product-header-card">
        <div class="product-bank-logo">
          <img :src="getBankLogo(product.kor_co_nm)" :alt="product.kor_co_nm + ' 로고'" />
        </div>

        <div class="product-main-info">
          <div class="product-badges">
            <span class="product-type-badge">{{ getProductTypeName(product.product_type) }}</span>
            <span v-if="isNew" class="new-badge">신규</span>
          </div>

          <h1 class="product-name">{{ product.fin_prdt_nm }}</h1>

          <div class="product-bank">
            <span>{{ product.kor_co_nm }}</span>
            <button @click="visitBankWebsite" class="visit-bank-button">
              홈페이지 방문 <i class="fas fa-external-link-alt"></i>
            </button>
          </div>

          <div class="product-join-methods">
            <JoinMethods :join_way="product.join_way" />
          </div>
        </div>

        <!-- Rate highlight section based on product type -->
        <div class="product-rate-highlight">
          <template v-if="product.product_type === 'deposit' || product.product_type === 'saving'">
            <RateDisplay
              :rate="product.max_rate"
              :type="product.product_type"
              rateType="max"
              :highlight="true"
            />
            <div class="rate-info">* 최고 우대금리 기준</div>
          </template>

          <template v-else-if="product.product_type === 'loan'">
            <RateDisplay :rate="product.min_rate" type="loan" rateType="min" :highlight="true" />
            <div class="rate-info">* 최저 금리 기준</div>
          </template>
        </div>
      </div>

      <!-- Product details tabs -->
      <div class="product-tabs">
        <button
          :class="['tab-button', { active: activeTab === 'details' }]"
          @click="activeTab = 'details'"
        >
          상품 정보
        </button>
        <button
          :class="['tab-button', { active: activeTab === 'rates' }]"
          @click="activeTab = 'rates'"
        >
          금리 정보
        </button>
        <button
          :class="['tab-button', { active: activeTab === 'conditions' }]"
          @click="activeTab = 'conditions'"
        >
          가입 조건
        </button>
      </div>

      <!-- Tab content -->
      <div class="tab-content">
        <!-- Details tab -->
        <div v-if="activeTab === 'details'" class="tab-panel">
          <div class="info-section">
            <h3 class="section-title">기본 정보</h3>

            <div class="info-grid">
              <div class="info-item">
                <div class="info-label">상품 코드</div>
                <div class="info-value">{{ product.fin_prdt_cd }}</div>
              </div>

              <div class="info-item">
                <div class="info-label">출시일</div>
                <div class="info-value">{{ formatDate(product.fin_prdt_launch_dt) }}</div>
              </div>

              <div class="info-item">
                <div class="info-label">금융 회사</div>
                <div class="info-value">{{ product.kor_co_nm }}</div>
              </div>

              <div class="info-item">
                <div class="info-label">가입 방법</div>
                <div class="info-value">
                  <JoinMethods :joinWay="product.join_way" :showLabel="false" />
                </div>
              </div>
            </div>
          </div>

          <div class="info-section">
            <h3 class="section-title">상품 설명</h3>
            <div class="product-description">
              {{ product.etc_note || '상세 설명이 제공되지 않은 상품입니다.' }}
            </div>
          </div>

          <!-- Product type-specific information -->
          <div v-if="product.product_type === 'saving'" class="info-section">
            <h3 class="section-title">적금 정보</h3>
            <div class="info-grid">
              <div class="info-item">
                <div class="info-label">적금 기간</div>
                <div class="info-value">
                  {{ product.save_trm ? product.save_trm + '개월' : '자유 적금' }}
                </div>
              </div>

              <div class="info-item">
                <div class="info-label">적립 유형</div>
                <div class="info-value">{{ getSavingType(product) }}</div>
              </div>
            </div>
          </div>

          <div v-if="product.product_type === 'loan'" class="info-section">
            <h3 class="section-title">대출 정보</h3>
            <div class="info-grid">
              <div class="info-item">
                <div class="info-label">대출 유형</div>
                <div class="info-value">{{ getLoanTypeName(product) }}</div>
              </div>

              <div v-if="product.loan_lmt" class="info-item">
                <div class="info-label">대출 한도</div>
                <div class="info-value">{{ formatCurrency(product.loan_lmt) }}</div>
              </div>
            </div>
          </div>
        </div>

        <!-- Rates tab -->
        <div v-if="activeTab === 'rates'" class="tab-panel">
          <template v-if="product.product_type === 'deposit' || product.product_type === 'saving'">
            <div class="info-section">
              <h3 class="section-title">금리 정보</h3>

              <div class="rates-grid">
                <div class="rate-card">
                  <div class="rate-card-title">기본금리</div>
                  <RateDisplay
                    :rate="product.base_rate"
                    :type="product.product_type"
                    rateType="base"
                  />
                </div>

                <div class="rate-card">
                  <div class="rate-card-title">최고금리</div>
                  <RateDisplay
                    :rate="product.max_rate"
                    :type="product.product_type"
                    rateType="max"
                    :highlight="true"
                  />
                </div>
              </div>

              <div class="info-item wide">
                <div class="info-label">이자 지급 방식</div>
                <div class="info-value">
                  {{ getInterestPaymentMethod(product.intr_rate_type_nm) }}
                </div>
              </div>

              <div class="disclaimer">
                * 실제 적용금리는 고객의 신용도, 대출기간, 대출금액 등에 따라 달라질 수 있습니다. *
                금리 우대조건은 은행 홈페이지에서 확인하시기 바랍니다.
              </div>
            </div>
          </template>

          <template v-else-if="product.product_type === 'loan'">
            <div class="info-section">
              <h3 class="section-title">대출 금리 정보</h3>

              <div class="rates-grid">
                <div class="rate-card">
                  <div class="rate-card-title">최저금리</div>
                  <RateDisplay
                    :rate="product.min_rate"
                    type="loan"
                    rateType="min"
                    :highlight="true"
                  />
                </div>

                <div class="rate-card">
                  <div class="rate-card-title">최고금리</div>
                  <RateDisplay :rate="product.max_rate" type="loan" rateType="max" />
                </div>
              </div>

              <div class="info-item wide">
                <div class="info-label">금리 유형</div>
                <div class="info-value">{{ getLoanRateType(product.lend_rate_type) }}</div>
              </div>

              <div class="disclaimer">
                * 실제 적용금리는 고객의 신용도, 대출기간, 대출금액 등에 따라 달라질 수 있습니다. *
                대출금리 우대조건은 은행 홈페이지에서 확인하시기 바랍니다.
              </div>
            </div>
          </template>
        </div>

        <!-- Conditions tab -->
        <div v-if="activeTab === 'conditions'" class="tab-panel">
          <div class="info-section">
            <h3 class="section-title">가입 조건</h3>

            <div class="conditions-list">
              <div class="condition-item">
                <i class="fas fa-check-circle"></i>
                <span>가입 가능 연령: 만 19세 이상</span>
              </div>

              <div class="condition-item">
                <i class="fas fa-check-circle"></i>
                <span>실명의 개인 (법인 및 임의단체 제외)</span>
              </div>

              <div
                v-if="product.product_type === 'deposit' || product.product_type === 'saving'"
                class="condition-item"
              >
                <i class="fas fa-check-circle"></i>
                <span
                  >가입 기간:
                  {{ product.save_trm ? product.save_trm + '개월' : '자유 입출금식' }}</span
                >
              </div>

              <div v-if="product.product_type === 'loan'" class="condition-item">
                <i class="fas fa-check-circle"></i>
                <span>대출 유형: {{ getLoanTypeName(product) }}</span>
              </div>
            </div>

            <div class="requirement-note">
              * 자세한 가입 요건은 해당 금융회사에 문의하시기 바랍니다.
            </div>
          </div>

          <div class="info-section">
            <h3 class="section-title">가입 방법</h3>

            <div class="join-methods-detailed">
              <template v-if="hasJoinWay('영업점')">
                <div class="join-method-item">
                  <div class="join-method-icon">
                    <i class="fas fa-building"></i>
                  </div>
                  <div class="join-method-content">
                    <h4>영업점 방문</h4>
                    <p>가까운 {{ product.kor_co_nm }} 지점을 방문하여 가입하실 수 있습니다.</p>
                  </div>
                </div>
              </template>

              <template v-if="hasJoinWay('인터넷')">
                <div class="join-method-item">
                  <div class="join-method-icon">
                    <i class="fas fa-laptop"></i>
                  </div>
                  <div class="join-method-content">
                    <h4>인터넷뱅킹</h4>
                    <p>
                      {{ product.kor_co_nm }} 인터넷뱅킹을 통해 온라인으로 가입하실 수 있습니다.
                    </p>
                  </div>
                </div>
              </template>

              <template v-if="hasJoinWay('스마트폰')">
                <div class="join-method-item">
                  <div class="join-method-icon">
                    <i class="fas fa-phone"></i>
                  </div>
                  <div class="join-method-content">
                    <h4>전화뱅킹</h4>
                    <p>{{ product.kor_co_nm }} 고객센터에 전화하여 가입하실 수 있습니다.</p>
                  </div>
                </div>
              </template>

              <template v-if="hasJoinWay('전화(텔레뱅킹)')">
                <div class="join-method-item">
                  <div class="join-method-icon">
                    <i class="fas fa-mobile-alt"></i>
                  </div>
                  <div class="join-method-content">
                    <h4>모바일뱅킹</h4>
                    <p>{{ product.kor_co_nm }} 모바일 앱을 통해 편리하게 가입하실 수 있습니다.</p>
                  </div>
                </div>
              </template>
            </div>
          </div>
        </div>
      </div>

      <!-- CTA section -->
      <div class="cta-section">
        <button @click="visitBankWebsite" class="cta-button primary">
          <i class="fas fa-external-link-alt"></i> {{ product.kor_co_nm }} 바로가기
        </button>
        <button @click="toggleFavorite" class="cta-button secondary">
          <i :class="isFavorite ? 'fas fa-heart' : 'far fa-heart'"></i>
          {{ isFavorite ? '즐겨찾기 해제' : '즐겨찾기 추가' }}
        </button>
      </div>

      <!-- Related products section (if available) -->
      <div v-if="relatedProducts.length > 0" class="related-products">
        <h3 class="section-title">관련 상품</h3>
        <div class="related-products-grid">
          <ProductCard
            v-for="relatedProduct in relatedProducts"
            :key="relatedProduct.id"
            :product="relatedProduct"
            :isFavorite="isProductInFavorites(relatedProduct.id)"
            @favorite-toggle="toggleRelatedFavorite"
          />
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import productsService from '@/services/products'
import RateDisplay from '@/components/products/RateDisplay.vue'
import JoinMethods from '@/components/products/JoinMethods.vue'
import ProductCard from '@/components/products/ProductCard.vue'

export default {
  name: 'ProductDetailView',
  components: {
    RateDisplay,
    JoinMethods,
    ProductCard,
  },
  setup() {
    // Router and route
    const route = useRoute()
    const router = useRouter()

    // Route params
    const productId = computed(() => route.params.id)
    const productType = computed(() => route.query.type || 'all')

    // State
    const product = ref(null)
    const loading = ref(true)
    const error = ref(null)
    const isFavorite = ref(false)
    const activeTab = ref('details')
    const relatedProducts = ref([])
    const favorites = ref([])

    // Computed props
    const isNew = computed(() => {
      if (!product.value || !product.value.fin_prdt_launch_dt) return false

      const launchDate = new Date(
        product.value.fin_prdt_launch_dt.substring(0, 4),
        product.value.fin_prdt_launch_dt.substring(4, 6) - 1,
        product.value.fin_prdt_launch_dt.substring(6, 8),
      )

      const threeMonthsAgo = new Date()
      threeMonthsAgo.setMonth(threeMonthsAgo.getMonth() - 3)

      return launchDate >= threeMonthsAgo
    })

    // Load product details
    const loadProductDetails = async () => {
      loading.value = true
      error.value = null

      try {
        // Load product based on type
        switch (productType.value) {
          case 'deposit':
            product.value = await productsService.getDepositProduct(productId.value)
            break
          case 'saving':
            product.value = await productsService.getSavingProduct(productId.value)
            break
          case 'loan':
            product.value = await productsService.getLoanProduct(productId.value)
            break
          default:
            product.value = await productsService.getFinancialProduct(productId.value)
            break
        }

        // Check if product is in favorites
        checkIfFavorite()

        // Load related products
        loadRelatedProducts()
      } catch (err) {
        console.error('Error loading product details:', err)
        error.value = '상품 정보를 불러오는데 실패했습니다. 다시 시도해주세요.'
      } finally {
        loading.value = false
      }
    }

    // Load related products
    const loadRelatedProducts = async () => {
      if (!product.value) return

      try {
        // Load products from same bank and same type
        let products = []

        switch (product.value.product_type) {
          case 'deposit':
            products = await productsService.getDepositProducts({
              bank: product.value.kor_co_nm,
            })
            break
          case 'saving':
            products = await productsService.getSavingProducts({
              bank: product.value.kor_co_nm,
            })
            break
          case 'loan':
            products = await productsService.getLoanProducts({
              bank: product.value.kor_co_nm,
            })
            break
        }

        // Filter out current product and limit to 3
        relatedProducts.value = products
          .filter((p) => p.id !== parseInt(productId.value))
          .slice(0, 3)
      } catch (err) {
        console.error('Error loading related products:', err)
      }
    }

    // Check if product is in user's favorites
    const checkIfFavorite = async () => {
      try {
        favorites.value = await productsService.getUserFavorites()
        isFavorite.value = favorites.value.some((fav) => {
          return fav.product === productId.value
        })
      } catch (err) {
        console.error('Error checking favorites:', err)
      }
    }

    // Check if a product is in favorites
    const isProductInFavorites = (id) => {
      return favorites.value.some((fav) => fav.product_id === id)
    }

    // Toggle favorite status
    const toggleFavorite = async () => {
      try {
        if (isFavorite.value) {
          await productsService.removeFromFavorites(productId.value)
        } else {
          await productsService.addToFavorites(productId.value)
        }
        isFavorite.value = !isFavorite.value
      } catch (err) {
        console.error('Error toggling favorite:', err)
      }
    }

    // Toggle favorite for related product
    const toggleRelatedFavorite = async (productId) => {
      try {
        if (isProductInFavorites(productId)) {
          await productsService.removeFromFavorites(productId)
        } else {
          await productsService.addToFavorites(productId)
        }

        // Refresh favorites
        await checkIfFavorite()
      } catch (err) {
        console.error('Error toggling related favorite:', err)
      }
    }

    // Format interest rate for display
    const formatRate = (rate) => {
      if (!rate) return '0.00'
      return parseFloat(rate).toFixed(2)
    }

    // Format date (YYYYMMDD to YYYY-MM-DD)
    const formatDate = (dateStr) => {
      if (!dateStr || dateStr.length !== 8) return '정보 없음'

      // Create Korean format date: YYYY년 MM월 DD일
      return `${dateStr.substring(0, 4)}년 ${dateStr.substring(4, 6)}월 ${dateStr.substring(6, 8)}일`
    }

    // Format currency amount
    const formatCurrency = (amount) => {
      if (!amount) return '한도 변동'
      return new Intl.NumberFormat('ko-KR', {
        style: 'currency',
        currency: 'KRW',
        maximumFractionDigits: 0,
      }).format(amount)
    }

    // Check if product has specific join way
    const hasJoinWay = (code) => {
      console.log(product.value)
      if (!product.value || !product.value.join_way) return false
      return product.value.join_way.includes(code)
    }
    
    // Get product type name for display
    const getProductTypeName = (type) => {
      const types = {
        deposit: '예금 상품',
        saving: '적금 상품',
        loan: '대출 상품',
      }
      return types[type] || '금융 상품'
    }

    // Get interest payment method name
    const getInterestPaymentMethod = (method) => {
      if (!method) return '명시되지 않음'

      if (method.includes('단리')) return '단리'
      if (method.includes('복리')) return '복리'

      return method
    }

    // Get saving type name
    const getSavingType = (saving) => {
      console.log(saving)
      // Implementation would depend on what data is available
      // This is a placeholder
      return '정기적금'
    }

    // Get loan rate type name
    const getLoanRateType = (type) => {
      const types = {
        1: '고정금리',
        2: '변동금리',
        3: '혼합금리',
      }
      return types[type] || '명시되지 않음'
    }

    // Get loan type name
    const getLoanTypeName = (loan) => {
      if (loan.is_mortgage) return '주택담보대출'
      if (loan.is_credit) return '신용대출'
      return '일반대출'
    }

    // Go back to products list
    const goBack = () => {
      router.push({ name: 'Products' })
    }

    // Share the current page
    const sharePage = () => {
      if (navigator.share) {
        navigator
          .share({
            title: product.value ? product.value.fin_prdt_nm : '금융 상품 정보',
            text: `${product.value ? product.value.kor_co_nm + '의 ' + product.value.fin_prdt_nm : '금융 상품'} 정보를 확인해보세요!`,
            url: window.location.href,
          })
          .catch((err) => console.error('Error sharing:', err))
      } else {
        // Fallback - copy URL to clipboard
        navigator.clipboard
          .writeText(window.location.href)
          .then(() => alert('URL이 클립보드에 복사되었습니다.'))
          .catch((err) => console.error('Error copying URL:', err))
      }
    }

    // Get bank logo
    const getBankLogo = (bankName) => {
      // This would be replaced with actual bank logo logic
      // For now, return a placeholder
      return `https://via.placeholder.com/50?text=${encodeURIComponent(bankName.charAt(0))}`
    }

    // Visit bank website
    const visitBankWebsite = () => {
      if (!product.value || !product.value.kor_co_nm) return

      // Simple mapping of bank names to websites
      const bankWebsites = {
        국민은행: 'https://www.kbstar.com',
        신한은행: 'https://www.shinhan.com',
        우리은행: 'https://www.wooribank.com',
        하나은행: 'https://www.kebhana.com',
        농협은행: 'https://www.nonghyup.com',
        기업은행: 'https://www.ibk.co.kr',
        산업은행: 'https://www.kdb.co.kr',
        새마을금고: 'https://www.kfcc.co.kr',
        수협은행: 'https://www.suhyup-bank.com',
        카카오뱅크: 'https://www.kakaobank.com',
        토스뱅크: 'https://www.tossbank.com',
        케이뱅크: 'https://www.kbanknow.com',
      }

      // Find exact or partial match
      const bankName = product.value.kor_co_nm
      let website = null

      // Try exact match first
      if (bankWebsites[bankName]) {
        website = bankWebsites[bankName]
      } else {
        // Try partial match
        for (const [bank, url] of Object.entries(bankWebsites)) {
          if (bankName.includes(bank)) {
            website = url
            break
          }
        }
      }

      // Open website or search
      if (website) {
        window.open(website, '_blank')
      } else {
        // If no match, search on Naver
        const searchQuery = encodeURIComponent(`${bankName} 공식 홈페이지`)
        window.open(`https://search.naver.com/search.naver?query=${searchQuery}`, '_blank')
      }
    }

    // Load data on component mount
    onMounted(() => {
      loadProductDetails()
    })

    return {
      product,
      loading,
      error,
      isFavorite,
      activeTab,
      relatedProducts,
      isNew,
      toggleFavorite,
      formatRate,
      formatDate,
      formatCurrency,
      hasJoinWay,
      getProductTypeName,
      getInterestPaymentMethod,
      getSavingType,
      getLoanRateType,
      getLoanTypeName,
      goBack,
      sharePage,
      visitBankWebsite,
      getBankLogo,
      isProductInFavorites,
      toggleRelatedFavorite,
      loadProductDetails,
    }
  },
}
</script>

<style scoped>
.product-detail-container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 1.5rem 1rem;
  font-family: 'Noto Sans KR', sans-serif;
  color: #333;
}

/* Loading animation */
.loading-container {
  text-align: center;
  padding: 5rem 0;
}

.loading-spinner {
  display: inline-block;
  width: 50px;
  height: 50px;
  border: 5px solid rgba(75, 85, 99, 0.1);
  border-radius: 50%;
  border-top-color: #3b82f6;
  animation: spin 1s ease-in-out infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Error container */
.error-container {
  text-align: center;
  padding: 4rem 2rem;
  background: #fef2f2;
  border-radius: 10px;
  color: #b91c1c;
}

.error-icon {
  font-size: 2.5rem;
  margin-bottom: 1rem;
}

.retry-button {
  margin-top: 1.5rem;
  padding: 0.5rem 1.5rem;
  background: #ef4444;
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}

.retry-button:hover {
  background: #dc2626;
}

/* Page header */
.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.back-button {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: #f3f4f6;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  color: #4b5563;
  font-size: 0.95rem;
  cursor: pointer;
  transition: all 0.2s;
}

.back-button:hover {
  background: #e5e7eb;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.share-button,
.favorite-button {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  color: #4b5563;
  font-size: 0.95rem;
  cursor: pointer;
  transition: all 0.2s;
}

.share-button:hover {
  background: #f9fafb;
}

.favorite-button.is-favorite {
  background: #fffbeb;
  border-color: #fbbf24;
  color: #d97706;
}

.favorite-button:hover {
  background: #f9fafb;
}

.favorite-button.is-favorite:hover {
  background: #fef3c7;
}

/* Product header card */
.product-header-card {
  display: grid;
  grid-template-columns: auto 1fr auto;
  gap: 1.5rem;
  padding: 1.5rem;
  background: white;
  border-radius: 10px;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.1),
    0 1px 2px rgba(0, 0, 0, 0.06);
  margin-bottom: 1.5rem;
  align-items: center;
}

.product-bank-logo {
  width: 60px;
  height: 60px;
  background: #f3f4f6;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.product-bank-logo img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.product-main-info {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.product-badges {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
}

.product-type-badge {
  font-size: 0.8rem;
  font-weight: 500;
  padding: 0.25rem 0.75rem;
  background: #e0f2fe;
  color: #0369a1;
  border-radius: 20px;
}

.new-badge {
  font-size: 0.8rem;
  font-weight: 500;
  padding: 0.25rem 0.75rem;
  background: #ecfdf5;
  color: #047857;
  border-radius: 20px;
}

.product-name {
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0;
  color: #111827;
}

.product-bank {
  display: flex;
  align-items: center;
  gap: 1rem;
  color: #4b5563;
  font-size: 0.95rem;
}

.visit-bank-button {
  font-size: 0.85rem;
  color: #2563eb;
  background: none;
  border: none;
  display: flex;
  align-items: center;
  gap: 0.25rem;
  cursor: pointer;
  padding: 0;
}

.visit-bank-button:hover {
  text-decoration: underline;
}

.product-join-methods {
  margin-top: 0.5rem;
}

.product-rate-highlight {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 8px;
  min-width: 150px;
}

.rate-info {
  font-size: 0.8rem;
  color: #64748b;
  margin-top: 0.5rem;
}

/* Tabs */
.product-tabs {
  display: flex;
  border-bottom: 1px solid #e5e7eb;
  margin-bottom: 1.5rem;
}

.tab-button {
  padding: 0.75rem 1.25rem;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  color: #6b7280;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.tab-button.active {
  color: #2563eb;
  border-bottom-color: #2563eb;
}

.tab-button:hover:not(.active) {
  color: #374151;
  background: #f9fafb;
}

/* Tab content */
.tab-panel {
  padding: 1rem 0;
}

.info-section {
  margin-bottom: 2rem;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  margin-bottom: 1.25rem;
  color: #111827;
  padding-bottom: 0.5rem;
  border-bottom: 1px solid #f3f4f6;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-item.wide {
  grid-column: 1 / -1;
}

.info-label {
  font-size: 0.9rem;
  color: #6b7280;
}

.info-value {
  font-size: 1rem;
  color: #111827;
}

.product-description {
  line-height: 1.6;
  color: #374151;
  white-space: pre-line;
}

.disclaimer {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #f9fafb;
  border-left: 3px solid #d1d5db;
  font-size: 0.85rem;
  color: #6b7280;
  line-height: 1.6;
  white-space: pre-line;
}

/* Rate cards */
.rates-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.rate-card {
  padding: 1.25rem;
  background: #f9fafb;
  border-radius: 8px;
  text-align: center;
}

.rate-card-title {
  font-size: 0.9rem;
  color: #6b7280;
  margin-bottom: 0.75rem;
}

/* Join method cards for conditions tab */
.join-methods-detailed {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.join-method-item {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  padding: 1.25rem;
  background: #f9fafb;
  border-radius: 8px;
}

.join-method-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 50px;
  height: 50px;
  background: #e0f2fe;
  color: #0284c7;
  border-radius: 50%;
  font-size: 1.25rem;
}

.join-method-content h4 {
  font-size: 1.1rem;
  font-weight: 600;
  margin: 0 0 0.5rem 0;
  color: #111827;
}

.join-method-content p {
  margin: 0;
  color: #4b5563;
  line-height: 1.5;
}

/* Condition items */
.conditions-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.condition-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  background: #f3f4f6;
  border-radius: 8px;
}

.condition-item i {
  color: #10b981;
  font-size: 1.1rem;
}

.requirement-note {
  font-size: 0.9rem;
  color: #6b7280;
  margin-top: 1rem;
}

/* CTA section */
.cta-section {
  display: flex;
  gap: 1rem;
  margin: 2rem 0;
}

.cta-button {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 1rem;
  border-radius: 8px;
  font-weight: 500;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s;
}

.cta-button.primary {
  background: #2563eb;
  color: white;
  border: none;
}

.cta-button.primary:hover {
  background: #1d4ed8;
}

.cta-button.secondary {
  background: white;
  color: #4b5563;
  border: 1px solid #e5e7eb;
}

.cta-button.secondary:hover {
  background: #f9fafb;
}

/* Related products */
.related-products {
  margin-top: 3rem;
  padding-top: 2rem;
  border-top: 1px solid #e5e7eb;
}

.related-products-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.25rem;
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .product-header-card {
    grid-template-columns: 1fr;
    gap: 1rem;
  }

  .product-bank-logo {
    justify-self: center;
  }

  .product-rate-highlight {
    justify-self: center;
    width: 100%;
  }

  .cta-section {
    flex-direction: column;
  }

  .info-grid {
    grid-template-columns: 1fr;
  }

  .rates-grid {
    grid-template-columns: 1fr;
  }
}
</style>
