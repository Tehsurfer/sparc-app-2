<template>
  <el-form
    ref="submitForm"
    label-position="top"
    :model="form"
    :rules="formRules"
    :hide-required-asterisk="true"
  >
    <el-form-item
      prop="shortDescription"
      label="Provide a short description of your research *"
    >
      <el-input
        v-model="form.shortDescription"
        placeholder="(Example: I am studying the effects of <approach> on <anatomical structure>)"
      />
    </el-form-item>

    <el-form-item prop="detailedDescription" label="Provide a detailed description of your research *">
      <el-input
        v-model="form.detailedDescription"
        type="textarea"
        :rows="3"
        placeholder="Please provide specifics about your research. Our curation team will then contact you."
      />
    </el-form-item>

    <el-form-item
      class="mt-32"
      prop="publishedManuscript"
      label="Has data been published in a manuscript? *"
    >
      <sparc-radio
        :value="form.publishedManuscript"
        @input="form.publishedManuscript = $event.target.value"
        label="Yes"
        display="Yes"
      />
      <sparc-radio
        :value="form.publishedManuscript"
        @input="form.publishedManuscript = $event.target.value"
        label="No"
        display="No"
      />
      <sparc-radio
        :value="form.publishedManuscript"
        @input="form.publishedManuscript = $event.target.value"
        label="Pending"
        display="Pending"
      />
    </el-form-item>

    <el-form-item
      class="mt-0 vertical-content"
      prop="manuscriptDoi"
      :disabled="form.publishedManuscript !== 'Yes'"
    >
      <url-input :disabled="form.publishedManuscript !== 'Yes'" v-model="form.manuscriptDoi" placeholder="Enter DOI URL"/>
    </el-form-item>

    <hr/>

    <user-contact-form-item v-model="form.user"/>

    <hr/>

    <div class="heading2">
      Please check the box to proceed
    </div>

    <el-form-item prop="recaptcha">
      <recaptcha-checkbox v-model="form.recaptcha" class="recaptcha my-16 pl-16"/>
    </el-form-item>

    <hr/>

    <el-form-item>
      <el-button class="primary" :disabled="isSubmitting" @click="onSubmit">
        Submit
      </el-button>
      <p v-if="hasError" class="error">
        An error has occurred, please try again.
      </p>
    </el-form-item>
  </el-form>
</template>

<script>
import { useMainStore } from '@/store/index'
import NewsletterMixin from '@/components/ContactUsForms/NewsletterMixin'
import RecaptchaMixin from '@/mixins/recaptcha/index'
import UserContactFormItem from '@/components/ContactUsForms/UserContactFormItem.vue'
import { saveForm, loadForm, populateFormWithUserData } from '~/utils/utils'
import UrlInput from '@/components/Url/UrlInput.vue'

export default {
  name: 'FeedbackForm',

  mixins: [NewsletterMixin, RecaptchaMixin],

  components: {
    UserContactFormItem,
    UrlInput
  },

  data() {
    const validateDoi = (rule, value, callback) => {
      if (this.form.publishedManuscript === 'Yes' && value === '') {
        callback(new Error('Please enter a DOI URL'))
      }
      callback()
    }
    return {
      form: {
        recaptcha: '',
        detailedDescription: '',
        shortDescription: '',
        publishedManuscript: '',
        manuscriptDoi: '',
        user: {
          typeOfUser: null,
          firstName: useMainStore().firstName,
          lastName: useMainStore().lastName,
          email: useMainStore().profileEmail,
          sendCopy: true,
          shouldFollowUp: true,
          shouldSubscribe: false
        }
      },
      isSubmitting: false,
      formRules: {
        user: {
          typeOfUser: [
            {
              required: true,
              message: 'Please select one',
              trigger: 'change'
            }
          ],
          email: [
            {
              required: true,
              message: 'Please enter your email',
              type: 'email',
              trigger: 'blur',
            }
          ],
          firstName: [
            {
              required: true,
              message: 'Please enter your first name',
              trigger: 'blur',
            }
          ],
          lastName: [
            {
              required: true,
              message: 'Please enter your last name',
              trigger: 'blur',
            }
          ]
        },

        shortDescription: [
          {
            required: true,
            message: 'Please enter a description',
            trigger: 'change'
          }
        ],

        detailedDescription: [
          {
            required: true,
            message: 'Please enter a description',
            trigger: 'change'
          }
        ],

        publishedManuscript: [
          {
            required: true,
            message: 'Please select one option',
            trigger: 'change'
          }
        ],

        manuscriptDoi: [
          {
            trigger: 'change',
            validator: validateDoi
          }
        ],

        recaptcha: [
          {
            required: true,
            message: 'Please check the box',
            trigger: 'change'
          }
        ]
      }
    }
  },

  mounted() {
    // Reset form fields when showing the form
    this.hasError = false
    this.$refs.submitForm.resetFields()
    const form = loadForm()
    if (form) {
      this.form = {
        ...this.form,
        ...form
      }
    }
    populateFormWithUserData(this.form, this.firstName, this.lastName, this.profileEmail)
  },

  methods: {
    /**
     * Send form to endpoint
     */
    async sendForm() {
      const config = useRuntimeConfig()
      this.isSubmitting = true
      const description = `
        <b>Short description:</b><br>${this.form.shortDescription}<br><br>
        <b>Detailed description:</b><br>${this.form.detailedDescription}<br><br>
        <b>Has a published manuscript:</b><br>${this.form.publishedManuscript === 'Yes' ? this.form.manuscriptDoi : this.form.publishedManuscript}<br><br>
        <b>What type of user are you?</b><br>${this.form.user.typeOfUser}<br><br>
        <b>Name:</b><br>${this.form.user.firstName} ${this.form.user.lastName}<br><br>
        <b>Email:</b><br>${this.form.user.email}
      `
      let formData = new FormData();
      formData.append("type", "research")
      formData.append("sendCopy", this.form.user.sendCopy)
      formData.append("title", `SPARC Research Submission: ${this.form.shortDescription}`)
      formData.append("description", description)
      formData.append("userEmail", this.form.user.email)

      // Save form to sessionStorage
      saveForm(this.form)

      await this.$axios
        .post(`${config.public.portal_api}/tasks`, formData)
        .then(() => {
          if (this.form.user.shouldSubscribe) {
            this.subscribeToNewsletter(this.form.user.email, this.form.user.firstName, this.form.user.lastName)
          } else {
            this.$emit('submit', this.form.user.firstName)
          }
        })
        .catch(() => {
          this.hasError = true
        })
        .finally(() => {
          this.isSubmitting = false
        })
    }
  },

  watch: {
    firstName() {
      this.form.user.firstName = this.firstName
    },
    lastName() {
      this.form.user.lastName = this.lastName
    },
    profileEmail() {
      this.form.user.email = this.profileEmail
    }
  }
}
</script>

<style lang="scss" scoped>
@import 'sparc-design-system-components-2/src/assets/_variables.scss';

hr {
  border-top: none;
  border-left: none;
  border-width: 2px;
  border-color: $lineColor1;
  margin: 2rem 0;
}
.error {
  color: $danger;
}
.recaptcha {
  display: flex;
  justify-content: left;
}
:deep(.el-form-item__content) {
  display: block;
}
</style>
