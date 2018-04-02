<template>
    <div v-title="'审核KYC'" class="kyc-audit-page">
        <div class="table-wrapper">
            <Progress :percent="currentPercent" :stroke-width="5"></Progress>
            <Table highlight-row
                   disabled-hover
                   border
                   :loading="isTableLoading"
                   :columns="columns"
                   :data="tableData"
                   @on-current-change="onSelect">
            </Table>
        </div>
        <Page :total="kycList.length"
              :current.sync="currentPageNumber"
              :page-size="MAX_LINE_IN_PAGE"
              show-total
              show-elevator>
        </Page>
        <div class="photo-preview-wrapper">
            <photo-preview :kycInfo="currentRow"></photo-preview>
        </div>
    </div>
</template>
<script>
    import moment from 'moment'
    import * as api from 'api/operate'
    import {getCountryByName} from 'common_libs/country_codes'
    import {KycState} from '../../../libs/kyc_const'
    import photoPreview from './photo_preview'

    const MAX_LINE_IN_PAGE = 20
    // 只拉取这种KYC状态的数据
    const FETCH_STATE = KycState.WAITING_AUDIT

    export default {
        name: 'kyc-audit-page',
        data() {
            return {
                // 拉下来的原始数据列表
                kycList: [],
                // 本次拉取到的所有待审核的数据条数，刷新页面后才刷新该数据
                totalKyc: 0,
                // 当前选中行的数据
                currentRow: {
                    photoIdDocument: '',
                    uploadSelfie: '',
                },
                // 当前表格页码，从1开始
                currentPageNumber: 1,
                // 是否正在加载表格数据
                isTableLoading: false,
                // 表格列信息
                columns: [
                    {title: '邮箱', key: 'email'},
                    {title: '国籍', key: 'country'},
                    {title: '身份证号', key: 'photoIdNumber', render: this.photoIdInputRender},
                    {title: '身份证照片', key: 'photoIdDocument', render: this.makeImgRender('photoIdDocument')},
                    {title: '自拍', key: 'uploadSelfie', render: this.makeImgRender('uploadSelfie')},
                    {title: '提交日期', key: 'commitDateStr'},
                    {title: '操作', key: 'action', render: this.opBtnsRender}
                ],
                MAX_LINE_IN_PAGE,
                FETCH_STATE,
            }
        },
        computed: {
            // 表格当前页面的数据
            tableData() {
                const startIndex = (this.currentPageNumber - 1) * MAX_LINE_IN_PAGE
                const result = this.kycList.slice(startIndex, startIndex + MAX_LINE_IN_PAGE)
                if (result.length) {
                    // eslint-disable-next-line no-underscore-dangle
                    result[0]._highlight = true
                    this.currentRow = result[0]
                    result.forEach(item => {
                        item.country = `[${getCountryByName(item.nationality)}] ${item.nationality}`
                        item.commitDateStr = moment(item.timestamp * 1000).format('YYYY/MM/DD hh:mm:ss')
                    })
                } else {
                    this.currentRow = {}
                }
                return result
            },
            // 当前选中行在当前页的行号
            currentRowIndex() {
                let index = -1
                this.tableData.some((item, i) => {
                    if (item.uin === this.currentRow.uin) {
                        index = i
                        return true
                    }
                    return false
                })
                return index
            },
            totalIndex() {
                if (this.currentRowIndex < 0) {
                    return -1
                }
                return (this.currentPageNumber - 1) * MAX_LINE_IN_PAGE + this.currentRowIndex
            },
            // 本次审核的进度
            currentPercent() {
                if (!this.totalKyc) {
                    return 100
                }
                return (this.totalKyc - this.kycList.length) / this.totalKyc * 100
            },
        },
        methods: {
            async changeKycState(uin, status) {
                const rowData = this.tableData.find(item => item.uin === uin)
                await api.updateKyc(uin, rowData.photoIdNumber, status, true)
                this.$Message.success(`${uin} 修改成功`)
                this.removeCurrentRow()
            },
            async updatePhotoId(uin, photoId) {
                const rowData = this.tableData.find(item => item.uin === uin)
                if (rowData && rowData.photoIdNumber === photoId) {
                    return
                }
                await api.updateKyc(uin, photoId, this.FETCH_STATE, false)
                this.$Message.success(`${uin} 修改成功`)
            },
            removeCurrentRow() {
                if (this.currentRowIndex >= 0) {
                    const isLastRow = this.kycList.length === (this.currentPageNumber - 1) * MAX_LINE_IN_PAGE + 1
                    this.kycList.splice(this.totalIndex, 1)
                    // 如果这一页的删光了，就展示上一页
                    if (isLastRow && this.currentPageNumber > 1) {
                        this.currentPageNumber--
                    }
                }
            },
            onSelect(currentRow) {
                this.currentRow = currentRow
            },
            photoIdInputRender(h, params) {
                return h('i-input', {
                    attrs: {value: params.row.photoIdNumber},
                    on: {
                        'on-blur': (event) => {
                            this.updatePhotoId(params.row.uin, event.target.value)
                        }
                    }
                })
            },
            makeImgRender(imgPropName) {
                return (h, params) => {
                    return h('img', {
                        attrs: {src: params.row[imgPropName]},
                    })
                }
            },
            opBtnsRender(h, params) {
                const successBtn = h('Button', {
                    props: {type: 'primary', size: 'small'},
                    style: {marginRight: '5px'},
                    on: {
                        click: () => {
                            this.changeKycState(params.row.uin, KycState.SUCCESS)
                        }
                    }
                }, '通过')
                const failBtn = h('Button', {
                    props: {type: 'error', size: 'small'},
                    on: {
                        click: () => {
                            this.changeKycState(params.row.uin, KycState.FAIL)
                        }
                    }
                }, '不通过')

                return h('div', {}, [successBtn, failBtn]);
            },
        },
        components: {
            photoPreview,
        },
        async mounted() {
            this.isTableLoading = true
            try {
                const data = await api.getKycList(this.FETCH_STATE)
                this.kycList = data || []
                this.totalKyc = this.kycList.length
            } catch (e) {
                console.log(e)
            }
            this.isTableLoading = false
        }
    }
</script>

<style lang="scss" rel="stylesheet/scss">
    @import '../../../../../common_assets/basic_const';

    .kyc-audit-page {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: space-between;
        .table-wrapper {
            padding: 10px 0 20px;
            overflow: auto;
            width: 100%;
            img {
                width: 50px;
                height: 50px;
            }
        }
        .photo-preview-wrapper {
            box-shadow: 0 -1px 4px 0 rgba(0, 0, 0, .2);
            width: 100%;
            height: 60vh;
            padding: 20px;
        }
    }
</style>