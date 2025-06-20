<script setup>
import { ref, onMounted } from "vue";
import { Cloudinary } from "@cloudinary/url-gen";
import { AdvancedImage } from "@cloudinary/vue";
import GallerySkeleton from "./GallerySkeleton.vue";

// Initialize Cloudinary instance
const cld = new Cloudinary({
  cloud: {
    cloudName: import.meta.env.VITE_CLOUDINARY_CLOUD_NAME,
  },
});

// State
const images = ref([]);
const error = ref(null);
const showSkeleton = ref(true);
// Dynamisk proxy-url beroende på miljö
const proxyUrl = import.meta.env.DEV
  ? "/api/cloudinary-proxy" // Lokalt via Vite
  : "https://photo-app-2.vercel.app/api/cloudinary-proxy"; // Vercel deploy

// Funktion för att hämta bilder från Cloudinary via proxy
const fetchImages = async () => {
  try {
    const response = await fetch(proxyUrl, {
      method: "POST",
    });
    const data = await response.json();

    if (!response.ok || !data.resources) {
      throw new Error(data.error?.message || "Kunde inte hämta bilder");
    }

    const processedImages = data.resources.map((resource) => {
      const lowQuality = cld
        .image(resource.public_id)
        .quality("auto:low")
        .format("jpg");

      const fullQualityUrl = cld.image(resource.public_id).toURL();

      return {
        id: resource.public_id,
        cldImg: lowQuality,
        fullUrl: fullQualityUrl,
      };
    });

    images.value.push(...processedImages);

    // Vänta tills alla bilder är färdigladdade i DOM
    const preloadPromises = processedImages.map((img) => {
      return new Promise((resolve, reject) => {
        const image = new Image();
        image.src = img.fullUrl;
        image.onload = resolve;
        image.onerror = reject;
      });
    });

    await Promise.all(preloadPromises);
  } catch (err) {
    error.value = err.message;
    console.error("Fel vid hämtning av bilder:", err);
  } finally {
    showSkeleton.value = false;
  }
};

const downloadImage = async (url) => {
  try {
    const response = await fetch(url);
    const blob = await response.blob();
    const blobUrl = URL.createObjectURL(blob);

    const link = document.createElement("a");
    link.href = blobUrl;
    link.download = "image.jpg"; // filnamn med .jpg
    link.type = blob.type; // lägg till MIME-typ

    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);

    // Vänta lite innan vi tar bort blob URL för att vara säker på att iOS hinner
    setTimeout(() => {
      URL.revokeObjectURL(blobUrl);
    }, 1000);
  } catch (err) {
    console.error("Fel vid nedladdning:", err);
  }
};

onMounted(() => {
  fetchImages();
});
</script>

<template>
  <div v-if="error" class="error">{{ error }}</div>

  <GallerySkeleton v-else-if="showSkeleton" />

  <div v-else class="masonry">
    <div v-for="img in images" :key="img.id" class="masonry-img-wrapper">
      <AdvancedImage :cldImg="img.cldImg" class="masonry-img" @load="$event.target.classList.add('loaded')"/>
      <div class="download-square" @click="downloadImage(img.fullUrl)">
        <span class="material-symbols-outlined">download</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Masonry layout using CSS columns */
.masonry {
  column-count: 3;
  column-gap: 1rem;
  padding: 1rem;
}

.masonry-img-wrapper {
  position: relative;
  margin-bottom: 1rem;
  break-inside: avoid;
}

/* Style for individual images */
.masonry-img {
  width: 100%;
  display: block;
  border-radius: 8px;
}

/* Download button styling */
.download-square {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 50px;
  height: 50px;
  color: black;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(20px);
  border-radius: 5px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
}

.download-square:hover {
  background: rgba(255, 255, 255, 0.4);
}

.masonry-img {
  transition: filter 0.3s ease;
  filter: blur(5px);
}

.masonry-img.loaded {
  filter: blur(0);
}

/* Responsive breakpoints */
@media screen and (max-width: 1000px) {
  .masonry {
    column-count: 2;
  }
}

@media screen and (max-width: 600px) {
  .masonry {
    column-count: 1;
  }
}

.error {
  color: red;
  padding: 16px;
}
</style>