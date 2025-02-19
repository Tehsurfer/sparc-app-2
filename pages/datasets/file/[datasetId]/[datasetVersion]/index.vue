<template>
  <div class="pb-32">
    <breadcrumb :breadcrumb="breadcrumb" :title="fileName" />
    <div class="container">
      <form ref="zipForm" method="POST" :action="zipitUrl">
        <input v-model="zipData" type="hidden" name="data" />
      </form>
      <content-tab-card
        v-if="hasViewer"
        class="mt-24"
        :tabs="tabs"
        :active-tab-id="activeTabId"
      >
        <biolucida-viewer
          v-if="hasBiolucidaViewer"
          v-show="activeTabId === 'imageViewer'"
          :data="biolucidaData"
          :datasetInfo="datasetInfo"
          :file="file"
        />
        <segmentation-viewer
          v-if="hasSegmentationViewer"
          v-show="activeTabId === 'segmentationViewer'"
          :data="segmentationData"
          :datasetInfo="datasetInfo"
          :file="file"
        />
        <plot-viewer
          v-if="hasPlotViewer"
          v-show="activeTabId === 'plotViewer'"
          :plotData="plotData"
          :datasetInfo="datasetInfo"
          :file="file"
        />
        <video-viewer
          v-if="hasVideoViewer"
          v-show="activeTabId === 'videoViewer'"
          :videoData="videoData"
          :videoSource="signedUrl"
          :datasetInfo="datasetInfo"
          :file="file"
        />
      </content-tab-card>
      <file-viewer-metadata
        v-if="!hasViewer"
        :datasetInfo="datasetInfo"
        :file="file"
        @download-file="executeDownload"
      />
    </div>
  </div>
</template>

<script>
import discover from '@/services/discover'
import biolucida from '@/services/biolucida'
import scicrunch from '@/services/scicrunch'
import BiolucidaViewer from '@/components/BiolucidaViewer/BiolucidaViewer'
import SegmentationViewer from '@/components/SegmentationViewer/SegmentationViewer'
import PlotViewer from '@/components/PlotViewer/PlotViewer'
import VideoViewer from '@/components/VideoViewer/VideoViewer'
import FileViewerMetadata from '@/components/ViewersMetadata/FileViewerMetadata.vue'
import FormatDate from '@/mixins/format-date'
import FetchPennsieveFile from '@/mixins/fetch-pennsieve-file'
import FileDetails from '@/mixins/file-details'

import { extractS3BucketName } from '@/utils/common'

import { isEmpty, pathOr, propOr } from 'ramda'

export default {
  name: 'DatasetFileDetailPage',

  components: {
    BiolucidaViewer,
    SegmentationViewer,
    PlotViewer,
    VideoViewer,
    FileViewerMetadata
  },

  mixins: [
    FormatDate,
    FetchPennsieveFile,
    FileDetails
  ],

  async setup() {
    const router = useRouter()
    const route = useRoute()
    const config = useRuntimeConfig()
    const { $axios, $pennsieveApiClient } = useNuxtApp()
    const url = `${config.public.discover_api_host}/datasets/${route.params.datasetId}`
    var datasetUrl = route.params.datasetVersion ? `${url}/versions/${route.params.datasetVersion}` : url
    let datasetInfo = {}
    await $pennsieveApiClient.value.get(datasetUrl).catch(e => {
      console.log(`Could not get the dataset's info: ${e}`)
    }).then(({ data }) => {
      datasetInfo = data
    })
    const s3Bucket = datasetInfo ? extractS3BucketName(datasetInfo.uri) : undefined
    const filePath = route.query.path   
    const file = await FetchPennsieveFile.methods.fetchPennsieveFile(
      filePath,
      route.params.datasetId,
      route.params.datasetVersion
    )

    // We should just be able to do as below and pull the source package id from file, but there are sometimes discrepancies between the pennsieve file sourcePackageId and the biolucida image data sourcePackageId returned from sparc.biolucida.net
    // const sourcePackageId = file.sourcePackageId
    // So now we must pull all the images from the dataset, then get each ones dataset info (to use the file name to map it) so that we can get the source package id from the right image 
    let sourcePackageId = ""
    const biolucidaSearchResults = await biolucida.searchDataset(route.params.datasetId)
    const imagesData = biolucidaSearchResults['dataset_images']
    if (imagesData != undefined) {
      await Promise.all(imagesData.map(async image => {
        const imageInfo = await biolucida.getImageInfo(image.image_id)
        if (imageInfo['name'] == file.name)
        {
          sourcePackageId = image['sourcepkg_id']
          return
        }
      }))
    }

    let biolucidaData = {}
    try {
      if (sourcePackageId != "") {
        await $axios.get(`${config.public.BL_API_URL}imagemap/sharelink/${sourcePackageId}/${route.params.datasetId}`).then(({ data }) => {
          biolucidaData = data
        })
      }
    } catch(e) {
      console.log(`Error retrieving biolucida data (possibly because there is none for this file): ${e}`)
    }
    const hasBiolucidaViewer = !isEmpty(biolucidaData) && biolucidaData.status !== 'error'

    // We must remove the N: in order for scicrunch to realize the package
    sourcePackageId = file.sourcePackageId
    const expectedScicrunchIdentifier = sourcePackageId != "" ? sourcePackageId.replace("N:", "") : ""
    let scicrunchData = {}
    try {
      if (expectedScicrunchIdentifier != "") {
        const scicrunchResponse = await scicrunch.getDatasetInfoFromObjectIdentifier(expectedScicrunchIdentifier)
        const result = pathOr([], ['data', 'result'], scicrunchResponse)
        scicrunchData = result?.length > 0 ? result[0] : []
      }
    } catch(e) {
      console.log(`Error retrieving sci crunch data (possibly because there is none for this file): ${e}`)
    }

    let segmentationData = {}
    // We should just be able to just pull from scicrunch response as shown below, but due to discrepancies we pull from the sparc api endpoint
    // const matchedSegmentationData = scicrunchData['mbf-segmentation']?.filter(function(el) {
    //   return el.identifier == expectedScicrunchIdentifier
    // })
    // segmentationData = segmentationData?.length > 0 ? matchedSegmentationData[0] : {}*/
    try {
      await discover.getSegmentationInfo(route.params.datasetId, filePath, s3Bucket).then(({ data }) => {
        segmentationData = data
      })
    } catch(e) {
      console.log(`Error retrieving segmentation data (possibly because there is none for this file): ${e}`)
    }
    const hasSegmentationViewer = !isEmpty(segmentationData)
    
    let plotData = {}
    const matchedPlotData = scicrunchData['abi-plot']?.filter(function(el) {
      return el.identifier == expectedScicrunchIdentifier
    })
    plotData = matchedPlotData?.length > 0 ? matchedPlotData[0] : {}
    const hasPlotViewer = !isEmpty(plotData)

    let videoData = {}
    const matchedVideoData = scicrunchData['video']?.filter(function(el) {
      return el.identifier == expectedScicrunchIdentifier
    })
    videoData = matchedVideoData?.length > 0 ? matchedVideoData[0] : {}
    const hasVideoViewer = !isEmpty(videoData)
    let signedUrl = ""
    if (hasVideoViewer) {
      const videoConfig = {
        params: {
          key: `${route.params.datasetId}/${filePath}`,
          contentType: videoData.mimetype.name,
          s3BucketName: s3Bucket
        }
      }
      signedUrl = await $axios.get(
          `${config.public.portal_api}/download`,
          videoConfig
        )
        .then(({ data }) => {
          return data
        })
    }

    let packageType = "None"
    if (sourcePackageId !== 'details') {
      packageType = file.packageType
    }
    const hasTimeseriesViewer = packageType === 'TimeSeries'
    if (hasTimeseriesViewer && config.public.SHOW_TIMESERIES_VIEWER) {
      router.push(`/datasets/timeseriesviewer?dataset_id=${route.params.datasetId}&dataset_version=${route.params.datasetVersion}&file_path=${filePath}`)
    }

    let activeTabId = hasBiolucidaViewer ? 'imageViewer' :
      hasTimeseriesViewer ? 'timeseriesViewer' :
      hasSegmentationViewer ? 'segmentationViewer' : 
      hasPlotViewer ? 'plotViewer' :
      hasVideoViewer ? 'videoViewer' : ''

    return {
      biolucidaData,
      videoData,
      plotData,
      segmentationData: {
        share_link: `${config.public.NL_LINK_PREFIX}/dataviewer?datasetId=${route.params.datasetId}&version=${route.params.datasetVersion}&path=${filePath}`,
        status: ''
      },
      file,
      hasBiolucidaViewer,
      hasPlotViewer,
      hasVideoViewer,
      hasSegmentationViewer,
      sourcePackageId,
      signedUrl,
      packageType,
      activeTabId,
      datasetInfo
    }
  },

  data: () => {
    const config = useRuntimeConfig()
    return {
      biolucidaData: {
        biolucida_image_id: '',
        share_link: '',
        status: ''
      },
      tabs: [],
      file: {},
      zipData: '',
      zipitUrl: config.public.zipit_api_host
    }
  },

  computed: {
    hasViewer: function() {
      return this.hasBiolucidaViewer || this.hasSegmentationViewer || this.hasPlotViewer || this.hasVideoViewer
    },
    datasetId: function() {
      return this.$route.params.datasetId
    },
    readme: function() {
      return propOr('', 'readme', this.datasetInfo)
    },
    breadcrumb: function() {
      return [
        {
          to: {
            name: 'index'
          },
          label: 'Home'
        },
        {
          to: {
            name: 'data'
          },
          label: 'Data & Models'
        },
        {
          to: {
            name: 'datasets-datasetId',
            params: {
              datasetId: this.datasetInfo.id
            },
          },
          label: `${this.datasetInfo.name}`
        },
        {
          to: {
            name: 'datasets-datasetId',
            params: {
              datasetId: this.datasetInfo.id
            },
            query: {
              datasetDetailsTab: 'files',
              path: `${this.fileFolderLocation}`
            }
          },
          label: 'File'
        }
      ]
    }
  },

  watch: {
    hasBiolucidaViewer: {
      handler: function(hasViewer) {
        if (hasViewer) {
          this.tabs.push({
            label: 'Biolucida Viewer',
            id: 'imageViewer'
          })
        } else {
          this.tabs = this.tabs.filter(tab => tab.id !== 'imageViewer')
        }
      },
      immediate: true
    },
    hasPlotViewer: {
      handler: function(hasViewer) {
        if (hasViewer) {
          this.tabs.push({
            label: 'Plot Viewer',
            id: 'plotViewer'
          })
        } else {
          this.tabs = this.tabs.filter(tab => tab.id !== 'plotViewer')
        }
      },
      immediate: true
    },
    hasVideoViewer: {
      handler: function(hasViewer) {
        if (hasViewer) {
          this.tabs.push({
            label: 'Video Viewer',
            id: 'videoViewer'
          })
        } else {
          this.tabs = this.tabs.filter(tab => tab.id !== 'videoViewer')
        }
      },
      immediate: true
    },
    hasSegmentationViewer: {
      handler: function(hasViewer) {
        if (hasViewer) {
          this.tabs.push({
            label: 'Segmentation Viewer',
            id: 'segmentationViewer'
          })
        } else {
          this.tabs = this.tabs.filter(tab => tab.id !== 'segmentationViewer')
        }
      },
      immediate: true
    }
  },

  methods: {
    executeDownload(file) {
      const version = this.$route.params.datasetVersion
      const datasetVersionRegexp = /(?<datasetId>\d*)\/(?<filePath>.*)/
      let params = file.uri.replace("s3://", "")
      let firstIndex = params.indexOf("/") + 1
      params = params.substr(firstIndex)
      const matches = params.match(datasetVersionRegexp)

      const payload = {
        paths: [matches.groups.filePath],
        datasetId: matches.groups.datasetId,
        version: version
      }

      this.zipData = JSON.stringify(payload, undefined)
      this.$nextTick(() => {
        this.$refs.zipForm.submit() // eslint-disable-line no-undef
      })
    },
    /**
     * Checks if file is MS Word, MS Excel, or MS Powerpoint
     * @param {Object} file
     */
    isMicrosoftFileType: function(file) {
      return (
        file.fileType == 'MSWord' ||
        file.fileType == 'MSExcel' ||
        file.fileType == 'PowerPoint'
      )
    },
  }
}
</script>
