
import { ElMessage, ElMessageBox } from 'element-plus'

import type { ErrorMessageMode } from '#/axios'

export function checkStatus(
  status: number,
  msg: string,
  errorMessageMode: ErrorMessageMode = 'message'
) {
  let errMessage = ''

  switch (status) {
    case 400:
      errMessage = `${msg}`
      break
    case 401:
      errMessage = msg
      break
    case 403:
      errMessage = '暂无权限查看'
      break
    case 404:
      errMessage = '网络请求错误，未找到该资源'
      break
    case 500:
      errMessage = '服务器错误，请联系管理员'
      break
    case 503:
      errMessage = '服务不可用，请稍候再试'
      break
    default:
      errMessage = `${msg}`
  }

  if (errMessage) {
    if (errorMessageMode === 'modal') {
      ElMessageBox.alert(errMessage, '提示', {
        confirmButtonText: '确定',
      })
    } else {
      ElMessage({
        message: errMessage,
        type: 'error',
      })
    }
  }
}
