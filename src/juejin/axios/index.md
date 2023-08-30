```ts
import axios from 'axios'
import { ElMessage, ElMessageBox } from 'element-plus'
import qs from 'qs'

import type { AxiosResponse } from 'axios'
import type { RequestOptions, Result, Recordable } from '#/axios'
import { useLoginStoreWithOut } from '@/stores/modules/user/login'
import type { AxiosTransform, CreateAxiosOptions } from './axiosTransform'

import OriginAxios from './originAxios'
import { checkStatus } from './checkStatus'

import { getToken } from '@/utils/auth'
import { isString } from '@/utils/is'

/**
 * @description: 数据处理，方便区分多种处理方式
 */
const transform: AxiosTransform = {
  /**
   * @description: 自定义的响应处理钩子函数，处理响应数据。如果数据不是预期格式，可直接抛出错误
   */
  transformResponseHook: (
    res: AxiosResponse<Result>,
    options?: RequestOptions
  ) => {
    if (!res) {
      return
    }
    const { isTransformResponse, isReturnNativeResponse } = options || {}
    // 是否返回原生响应头 比如：需要获取响应头时使用该属性
    if (isReturnNativeResponse) {
      return res
    }
    // 不进行任何处理，直接返回
    // 用于页面代码可能需要直接获取code，data，message这些信息时开启
    if (!isTransformResponse) {
      return res.data
    }
    // 错误的时候返回

    const { data = { code: '', message: '' } } = res || {}
    if (!data) {
      // return '[HTTP] Request has no return value';
      throw new Error('请求失败，请稍后重试')
    }
    //  这里 code，result，message为 后台统一的字段，需要在 types.ts内修改为项目自己的接口返回格式
    // 这里逻辑可以根据项目进行修改
    // const hasSuccess = data && Reflect.has(data, 'code') && code === 0
    if (res?.status === 200 && data) {
      return data
    }

    // 在此处根据自己项目的实际情况对不同的code执行不同的操作
    // 如果不希望中断当前请求，请return数据，否则直接抛出异常即可
    let timeoutMsg = ''
    switch (data.code) {
      case 401:
        timeoutMsg = '登录超时,请重新登录!'
        const useLoginStore = useLoginStoreWithOut()
        useLoginStore.setToken('')
        useLoginStore.logout()
        break
      default:
        if (data.message) {
          timeoutMsg = data.message
        }
    }

    // errorMessageMode='modal'的时候会显示modal错误弹窗，而不是消息提示，用于一些比较重要的错误
    // errorMessageMode='none' 一般是调用时明确表示不希望自动弹出错误提示
    if (options?.errorMessageMode === 'modal') {
      ElMessageBox({ title: '错误提示', message: timeoutMsg })
    } else if (options?.errorMessageMode === 'message') {
      ElMessage.error(timeoutMsg)
    }

    throw new Error(timeoutMsg || '请求出错，请稍候重试')
  },

  // 自定义的请求处理钩子函数，请求之前处理config
  beforeRequestHook: (config, options = {}) => {
    const { apiUrl, joinPrefix, joinParamsToUrl, urlPrefix } = options

    if (joinPrefix) {
      config.url = `${urlPrefix}${config.url}`
    }

    if (apiUrl && isString(apiUrl)) {
      config.url = `${apiUrl}${config.url}`
    }
    const params = config.params || {}
    const data = config.data || false
    if (config.method?.toUpperCase() === 'GET') {
      if (!isString(params)) {
        config.params = Object.assign(params || {})
      } else {
        // 兼容restful风格
        config.url = config.url + params
        config.params = undefined
      }
    } else {
      if (!isString(params)) {
        if (
          Reflect.has(config, 'data') &&
          config.data &&
          (Object.keys(config.data).length > 0 ||
            config.data instanceof FormData)
        ) {
          config.data = data
          config.params = params
        } else {
          // 非GET请求如果没有提供data，则将params视为data
          config.data = params
          config.params = undefined
        }
        if (joinParamsToUrl) {
          config.url =
            config.url +
            qs.stringify(Object.assign({}, config.params, config.data), {
              addQueryPrefix: true,
            })
        }
      } else {
        // 兼容restful风格
        config.url = config.url + params
        config.params = undefined
      }
    }
    return config
  },

  /**
   * @description: 请求拦截器处理
   */
  requestInterceptors: (config, options) => {
    // 添加请求头Authorization
    const token = getToken()
    if (token && (config as Recordable)?.requestOptions?.withToken !== false) {
      // jwt token
      ;(config as Recordable).headers.Authorization =
        options.authenticationScheme
          ? `${options.authenticationScheme} ${token}`
          : token
    }
    return config
  },

  /**
   * @description: 响应拦截器处理
   */
  responseInterceptors: (res: AxiosResponse<any>) => {
    return res
  },

  /**
   * @description: 响应错误处理
   */
  responseInterceptorsCatch: (error?: any) => {
    const { response, code, message, config } = error || {}
    const errorMessageMode = config?.requestOptions?.errorMessageMode || 'none'
    let errMessage: string = response?.data?.message ?? ''
    const err: string = error?.toString?.() ?? ''

    if (axios.isCancel(error)) {
      return Promise.reject(error)
    }

    try {
      if (code === 'ECONNABORTED' && message.indexOf('timeout') !== -1) {
        errMessage = '接口请求超时,请刷新页面重试!'
      }
      if (err?.includes('Network Error')) {
        errMessage = '网络异常，请检查您的网络连接是否正常!'
      }

      if (errMessage) {
        if (errorMessageMode === 'modal') {
          ElMessageBox({ title: '错误提示', message: errMessage })
        } else if (errorMessageMode === 'message') {
          ElMessage.error(errMessage)
        }
        return Promise.reject(error)
      }
    } catch (error) {
      throw new Error(error as unknown as string)
    }

    checkStatus(error?.response?.status, errMessage, errorMessageMode)
    return Promise.reject(error)
  },
}

function createAxios(opt?: Partial<CreateAxiosOptions>) {
  return new OriginAxios({
    ...{
      authenticationScheme: 'Bearer',
      baseURL: import.meta.env.VITE__APP_BASE_URL,
      withCredentials: true,
      // 10 秒超时
      timeout: 10 * 1000,
      headers: {
        'Content-Type': 'application/json;charset=UTF-8',
        Accept: 'application/json;charset=utf-8',
      },
      transform,
      requestOptions: {
        // 默认给url添加前缀
        joinPrefix: true,
        // 是否返回原生响应头 比如：需要获取响应头时使用该属性
        isReturnNativeResponse: false,
        // 需要对返回数据进行处理
        isTransformResponse: true,
        // post请求的时候添加参数到url
        joinParamsToUrl: false,
        // 格式化提交参数时间
        formatDate: true,
        // 消息提示类型
        errorMessageMode: 'message',
        // 接口地址
        // apiUrl: import.meta.env.VITE_APP_BASE_API,
        // 接口拼接地址
        urlPrefix: '',
        //  是否加入时间戳
        joinTime: false,
        // 忽略重复请求
        ignoreCancelToken: true,
        // 是否携带token
        withToken: true,
        retryRequest: {
          isOpenRetry: true,
          count: 5,
          waitTime: 100,
        },
      },
    },
    ...opt,
  })
}

export const http = createAxios()

// 用法
// import { http } from '@/utils/axios'
// export function getProvinceList() {
//   return http.request<BasicResponse<Array<IOption>>>({
//     url: '/open/enterprise/provinces',
//     method: 'get',
//   })
// }
```
