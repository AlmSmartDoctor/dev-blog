## Presentation Layer

#### View
- 데이터가 사용자에게 어떻게 표시되는지를 나타냄

#### Presenter
- View가 어떤 식으로 데이터를  보여주는 지는 몰라도 됨
- 테스트에 용이
- View concrete class 가 아니라 View Interface를 사용

```Csharp
using System.Collections.Generic;
using EntityLayer.Prescription;

namespace PresentationLayer.View.Prescription.GroupOrderSetting
{
    interface IPopupGroupOrderSettingView
    {
        IList<GroupOrder> GroupOrders { get; set; }
        IList<GroupOrderDetail> GroupOrderDetails { get; set; }
        GroupOrder SelectedGroupOrder { get; }
        GroupOrderDetail SelectedGroupOrderDetail { get; }
    }
}


```
IPopupGroupOrderSettingView.cs

```CSharp
using PresentationLayer.Presenter.Prescription.GroupOrder;

namespace PresentationLayer.View.Prescription.GroupOrderSetting
{
    public partial class PopupGroupOrderSettingView : XtraForm, IPopupGroupOrderSettingView
    {
        private readonly PopupGroupOrderSettingPresenter _presenter;

        public IList<GroupOrder> GroupOrders
        {
            get => (IList<GroupOrder>)gridControlGroupOrder.DataSource;
            set => gridControlGroupOrder.DataSource = value;
        }

        public IList<GroupOrderDetail> GroupOrderDetails
        {
            get => (IList<GroupOrderDetail>)gridControlGroupOrderDetail.DataSource;
            set => gridControlGroupOrderDetail.DataSource = value;
        }

        public GroupOrder SelectedGroupOrder
        {
            get => gridViewGroupOrder.GetFocusedRow() as GroupOrder;
        }

        public GroupOrderDetail SelectedGroupOrderDetail
        {
            get => gridViewGroupOrderDetail.GetFocusedRow() as GroupOrderDetail;
        }

        public PopupGroupOrderSettingView()
        {
            InitializeComponent();
            _presenter = new PopupGroupOrderSettingPresenter(this);
            _presenter.LoadData();
        }

        private void simpleButtonTutorial_Click(object sender, System.EventArgs e)
        {
            _presenter.OpenTutorial();
        }

        private void simpleButtonUpGroupOrderDetail_Click(object sender, System.EventArgs e)
        {
            _presenter.LiftUpGroupOrderDetail();
            gridViewGroupOrderDetail.RefreshData();
        }

        private void simpleButtonDownGroupOrderDetail_Click(object sender, System.EventArgs e)
        {
            _presenter.DropDownGroupOrderDetail();
            gridViewGroupOrderDetail.RefreshData();
        }

        private void gridViewGroupOrder_FocusedRowChanged(object sender, DevExpress.XtraGrid.Views.Base.FocusedRowChangedEventArgs e)
        {
            _presenter.LoadGroupOrderDetail();
        }

        private void simpleButtonSaveGroupOrderDetail_Click(object sender, System.EventArgs e)
        {
            _presenter.SaveGroupOrderDetail();

            MessageBox.Show(@"저장이 완료되었습니다.");
        }

        private void simpleButtonAddGroupOrder_Click(object sender, System.EventArgs e)
        {
            try
            {
                _presenter.AddGroupOrder(textEditGroupOrderName.Text.Trim());
            }
            catch (SameNameAlreadyExistsException exception)
            {
                MessageBox.Show(exception.Message);
            }

            _presenter.LoadGroupOrder();
        }

        private void simpleButtonAddGroupOrderDetail_Click(object sender, System.EventArgs e)
        {
            _presenter.AddGroupOrderDetail();
        }
    }
}
```
PopupGroupOrderSettingView.cs

```CSharp
using System.Collections;
using System.Collections.Generic;
using System.Data;
using System.Diagnostics;
using System.Linq;
using BusinessLogicLayer.Service.Prescription;
using Exceptions;
using PresentationLayer.Old;
using PresentationLayer.View.Prescription.GroupOrderSetting;

namespace PresentationLayer.Presenter.Prescription.GroupOrder
{
    internal class PopupGroupOrderSettingPresenter
    {
        private readonly IPopupGroupOrderSettingView _view;

        internal PopupGroupOrderSettingPresenter(IPopupGroupOrderSettingView view)
        {
            _view = view;
        }

        internal void LoadData()
        {
            LoadGroupOrder();
            LoadGroupOrderDetail();
        }

        internal void LoadGroupOrder()
        {
            _view.GroupOrders = GroupOrderService.Instance.ListGroupOrders();
        }

        internal void LoadGroupOrderDetail()
        {
            if (_view.SelectedGroupOrder == null) return;

            _view.GroupOrderDetails = GroupOrderService.Instance.ListGroupOrderDetails(_view.SelectedGroupOrder);
        }

        internal void OpenTutorial()
        {
            try
            {
                Process.Start("http://blog.naver.com/chunneung/221044511367");
            }
            catch
            {
                // ignored
            }
        }

        internal void LiftUpGroupOrderDetail()
        {
            if (_view.SelectedGroupOrderDetail == null) return;
            if (_view.SelectedGroupOrderDetail.SortNumber == 0) return;

            _view.SelectedGroupOrderDetail.SortNumber -= 1;
            _view.GroupOrderDetails.Single(g => g.SortNumber == _view.SelectedGroupOrderDetail.SortNumber && !g.Equals(_view.SelectedGroupOrderDetail)).SortNumber += 1;
        }

        internal void DropDownGroupOrderDetail()
        {
            if (_view.SelectedGroupOrderDetail == null) return;
            if (_view.SelectedGroupOrderDetail.SortNumber == _view.GroupOrderDetails.Count - 1) return;

            _view.SelectedGroupOrderDetail.SortNumber += 1;
            _view.GroupOrderDetails.Single(g => g.SortNumber == _view.SelectedGroupOrderDetail.SortNumber && !g.Equals(_view.SelectedGroupOrderDetail)).SortNumber -= 1;
        }

        internal void SaveGroupOrderDetail()
        {
            if (_view.SelectedGroupOrder == null) return;

            GroupOrderService.Instance.SaveGroupOrderDetail(_view.SelectedGroupOrder, _view.GroupOrderDetails);
        }

        internal void AddGroupOrder(string name)
        {
            if (_view.GroupOrders.Any(g => g.Name == name))
                throw new SameNameAlreadyExistsException(@"동일한 이름이 이미 존재합니다.");

            GroupOrderService.Instance.AddGroupOrder(name);
        }

        internal void AddGroupOrderDetail()
        {
            if (_view.SelectedGroupOrder == null) return;

            var param = new Hashtable();
            param.Add("ArtcCd", _view.SelectedGroupOrder.Code);
            param.Add("CdDtlCnt", _view.GroupOrderDetails.Count);
            param.Add("List", _view.GroupOrderDetails.Select(d => d.Code).ToList());

            var result = PopupHandler.popup(new PopupMdclPrscCdMgmt(), param);
            if (result != null && result.Contains("ArtcCd"))
                LoadGroupOrderDetail();
        }
    }
}

```
PopupGroupOrderSettingPresenter.cs

## BusinessLogic Layer
- Entity에 PrsentationLayer가 접근하지 못하게 하므로 Entity를 Presentation Layer에서 사용하는 클래스로 바꿔주는 로직
- 반대도 동일

```CSharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.RegularExpressions;
using DataAccessLayer;
using DataAccessLayer.Context;
using DataAccessLayer.Repository.BaseDataMgmt;
using EntityLayer.BaseDataMgmt;
using EntityLayer.Prescription;

namespace BusinessLogicLayer.Service.Prescription
{
    public class GroupOrderService
    {
        private static readonly Lazy<GroupOrderService> s_groupOrderService = new Lazy<GroupOrderService>(() => new GroupOrderService());

        private GroupOrderService() { }

        public static GroupOrderService Instance
        {
            get
            {
                return s_groupOrderService.Value;
            }
        }

        public IList<GroupOrder> ListGroupOrders()
        {
            using (var context = new MedicalDbContext())
            {
                var repository = new GroupOrderRepository(context);
                return repository.ListGroupOrderEntities()
                    .Select(e => new GroupOrder{ IsInserted = false, Code = e.Code, Name = e.Name })
                    .ToList();
            }
        }

        public IList<GroupOrderDetail> ListGroupOrderDetails(GroupOrder groupOrder)
        {
            using (var context = new MedicalDbContext())
            {
                var repository = new GroupOrderDetailRepository(context);
                return repository.ListPrescriptionsBelongToGroupOrder(groupOrder.Code)
                    .Select((prescriptionBelongToGroupOrder, index) =>
                        new GroupOrderDetail
                        {
                            SortNumber = index,
                            Code = prescriptionBelongToGroupOrder.Code,
                            Name = prescriptionBelongToGroupOrder.Name
                        })
                    .ToList();
            }
        }

        public void AddGroupOrder(string name)
        {
            using (var context = new MedicalDbContext())
            {
                var groupOrder = new GroupOrderEntity
                {
                    OrganizationId = Organization.Instance.Id,
                    Code = GetGroupOrderCode(),
                    Seqno = 1,
                    UserId = "0",
                    Name = name,
                    DisplayWay = "1",
                    SubjectCode = null,
                    SortNo = GetMaxSortNumber(),
                    RawGroupViewFlag = "N",
                    ValidFromDate = new DateTime(DateTime.Today.Year, 1, 1),
                    ValidToDate = new DateTime(2099, 12, 31),
                    GroupOrderCodeKind = "CRM",
                    SubGroupOrderCodeKind = null,
                    Memo = string.Empty,
                    Mx999 = string.Empty
                };

                context.GroupOrder.Add(groupOrder);
                context.SaveChanges();
            }
        }

        private static string GetGroupOrderCode()
        {
            const string basicCode = @"CRM_";

            using (var context = new MedicalDbContext())
            {
                var repository = new GroupOrderRepository(context);
                var entities = repository.ListGroupOrderEntities();

                var codeRegex = new Regex($"{basicCode}[0-9]+");

                int number = entities
                    .Select(e => codeRegex.Match(e.Code))
                    .Where(m => m.Success)
                    .Select(m => m.Value)
                    .DefaultIfEmpty()
                    .Max(s => string.IsNullOrEmpty(s) ? 0 : Convert.ToInt32(s.Replace(basicCode, string.Empty)));

                return $"{basicCode}{number}";
            }
        }

        private static int GetMaxSortNumber()
        {
            using (var context = new MedicalDbContext())
            {
                var repository = new GroupOrderRepository(context);
                return repository.GetMaxSortNumber();
            }
        }

        public void SaveGroupOrderDetail(GroupOrder groupOrder, IList<GroupOrderDetail> groupOrderDetails)
        {
            using (var context = new MedicalDbContext())
            {
                var repository = new GroupOrderDetailRepository(context);
                var entities = repository.ListGroupOrderDetailEntities(groupOrder.Code);

                foreach (var entity in entities)
                {
                    entity.SortNumber = groupOrderDetails.Single(g => g.Code == entity.DetailCode).SortNumber;
                }

                context.SaveChanges();
            }
        }
    }
}

```
GroupOrderService.cs


## Data Access Layer
- DbContext 및 기타 Select 쿼리를 저장하는 repository로 이루어짐
```CSharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using DataAccessLayer.Context;
using EntityLayer.BaseDataMgmt;
using EntityLayer.DataAccessObject.Model.Prescription;

namespace DataAccessLayer.Repository.BaseDataMgmt
{
    public class GroupOrderDetailRepository
    {
        private readonly MedicalDbContext _context;

        public GroupOrderDetailRepository(MedicalDbContext context)
        {
            _context = context;
        }

        public IList<GroupOrderDetailEntity> ListGroupOrderDetailEntities(string groupOrderCode)
        {
            return _context.GroupOrderDetail
                .Where(g => g.OrganizationId == Organization.Instance.Id &&
                            g.Code == groupOrderCode)
                .ToList();
        }

        public IList<PrescriptionBelongToGroupOrder> ListPrescriptionsBelongToGroupOrder(string groupOrderCode)
        {
            return _context.GroupOrderDetail
                .Where(g => g.OrganizationId == Organization.Instance.Id &&
                            g.Code == groupOrderCode)
                .Join(
                    _context.UsePrescriptionInformation,
                    g => new { OrganizationId = g.OrganizationId, PrescriptionCode = g.DetailCode },
                    p => new { OrganizationId = p.organizationId, PrescriptionCode = p.usePrescriptionCode },
                    (g, p) => new PrescriptionBelongToGroupOrder
                    {
                        SortNumber = g.SortNumber ?? 0,
                        Code = g.DetailCode,
                        Name = p.useName
                    })
                .OrderBy(p => p.SortNumber)
                .ToList();
        }
    }
}

```
GroupOrderDetailRepository.cs
