# How-is-the-advantage-of-using-PHP-Visibility
I want to understand much about PHP Visibility and it's advantages and disadvantages because i found more developer try to keep much of link code in a method instead of this case I have create more private method but they said "There are no advantages if just create unreusable method" so they still keep their code in a method is easy to read.
I've changed to new carrier with new young developers during working times we so seriously to talk about code structure and performance 
they try to keep they code is perfected (They keep more than 80 links of code in a method) and never except my suggestion. 
On the another hand I've read more time and tutorials and base on my experience I think I should used some method to wrap that code
by it's conditional and call it back but those method is not reusable I just separated the piece of code and make conditional 
for each method it may easy for me to validate errors and easy to implement but they said they difficult to click to find that 
method when they reach each conditional if keep as they structure may easy to read all of the link of code. 
I said I'm really nervously when I have try to read. And I've asked them This is OOP structures? They said Yes that is.

So today I want to know and make sure I've confused or they have not much experience?

Their function

public function postDisburse_old($loan_id = NULL)
    {
        $data = Request::except(['_token']);
        $rules = [
            'disburse_on' => 'required|date'
        ];
        $attribs = [
            'disburse_on' => 'Disburse Date',
        ];
        $validator = Validator::make($data, $rules);
        $validator->setAttributeNames($attribs);

        if ($validator->fails()) {
            return redirect()->back()->withErrors($validator);
        } else {

            $loan = Loan::with('schedule')->where('id', '=', $loan_id)->first();
            if (!empty($loan) && $loan->status == 2) {
                $loan->disburse_date = Request::input('disburse_on');
                $loan->contract_date = Request::input('contract_date');
                $loan->disburse_note = Request::input('note');
                $loan->start_date = Request::input('schedule_date');
                $loan->disburse_byuserid = Auth::user()->id;
                $loan->status = 3;

                // Update Schedule
                $holiday = [];
                if ($loan->holiday_flag == 1) {
                    $holiday = $loan->holiday;
                }
                $repayment_array = LoanCalculate::monthly_loan_schedule(
                    $loan->repayment_type, $loan->start_date, $loan->loan_duration, $loan->loan_amount, $loan->interest_rate, $loan->balloon, $loan->balloon_month,
                    $loan->monthly_payment, $loan->balloon_amount_array, $loan->custom_flag, $loan->days_of_month, $loan->holiday_flag, $holiday, $loan->schedule
                )[0];

                if (!empty($repayment_array)) {

                    $total_days = 0;
                    $total_interest = 0;
                    $total_principal = 0;
                    $total_monthly = 0;
                    $total_principal_bal = 0;

                    for ($i = 1; $i < count($repayment_array); $i++) {

                        $total_days = $total_days + $repayment_array[$i][1];
                        $total_interest += $repayment_array[$i][2];
                        $total_principal += $repayment_array[$i][3];
                        $total_monthly += $repayment_array[$i][4];
                        $total_principal_bal += $repayment_array[$i][5];
                    }

                    // Average Balance
                    $average_bal = $total_principal_bal / $loan->loan_duration;

                    if ($average_bal == 0 && $total_principal_bal == 0) {
                        Session::flash('message', 'Loan duration (Tenure) couldbe  not allow for one month');
                    } else {
                        $annual_yield = 100 * ($total_interest / $loan->loan_duration / $average_bal * 12);
                        $loan->annual_yield = $annual_yield;

                        if($loan->save()) {

                            $this->userActivity($loan->disburse_byuserid, $loan_id, 6, 'Disburse Loan');
                            RepaymentSchedule::where('loan_id', '=', $loan_id)->delete();
                            // Insert Repayment Schedule
                            for ($i = 1; $i < count($repayment_array); $i++) {
                                $data = ['loan_id' => $loan_id, 'schedule_date' => $repayment_array[$i][0], 'date_num' => $repayment_array[$i][1], 'interest' => $repayment_array[$i][2], 'principal' => $repayment_array[$i][3], 'intraday_rate' => $repayment_array[$i][6]];
                                RepaymentSchedule::insert($data);
                            }
                            $transaction = new TransactionsRequiry();
                            $transaction->id = TransactionsRequiry::select('id')->orderBy('id', 'desc')->first()->id + 1;
                            $transaction->loan_id = $loan_id;
                            $transaction->trans_date = date('Y-m-d');
                            $transaction->trans_type = "Disbursement";
                            $transaction->amount = $loan->loan_amount;
                            $transaction->description = $loan->disburse_note;
                            $transaction->balance = $loan->loan_amount;
                            $transaction->user_id = $loan->user_id;
                            $transaction->save();
                            // Update Client Loan Account
                            $loan_acc = ClientLoanAccounts::select(['id', 'activated_on', 'status','account_name'])->where('id', '=', $loan->loan_account_id)->first();
                            if (!empty($loan_acc) && count($loan_acc) > 0) {
                                $loan_acc->activated_on = date("Y-m-d");
                                $loan_acc->status = 2; // Activate(Std)
                                $loan_acc->balance = $loan->loan_amount;
                                if ($loan_acc->save()) {

                                } else {
                                    $loan->delete();
                                    $transaction->delete();
                                    Session::flash('message', 'Update client loan account information not successfully');
                                }
                            }
                            //Fee Charge and commission fee
                            $transaction1 = [];
                            if (Request::has('commission_fee') && Request::input('commission_fee') > 0) {
                                $fee_charge = new FeeCharge();
                                $fee_charge->charge_type = 4; // upfront charge
                                $fee_charge->charge_amount = floatval(Request::input('commission_fee')) * $transaction->amount / 100.0;
                                $fee_charge->charge_date = date('Y-m-d');
                                $fee_charge->receipt = Request::input('invoice_number');
                                $fee_charge->note = "Upfront charge for " . $loan->contract_id;
                                $fee_charge->save();

                                // Transaction
                                $transaction1 = new TransactionsRequiry();
                                $transaction1->id = TransactionsRequiry::select('id')->orderBy('id', 'desc')->first()->id + 1;
                                $transaction1->loan_id = $loan_id;
                                $transaction1->trans_date = date('Y-m-d');
                                $transaction1->trans_type = "Fee Charge Repayment";
                                $transaction1->amount = floatval(Request::input('commission_fee')) * $transaction->amount / 100.0;
                                $transaction1->fee = $transaction1->amount;
                                $transaction1->balance = $loan->loan_amount;
                                $transaction1->description = $fee_charge->note;
                                $transaction1->user_id = $loan->user_id;
                                if($transaction1->save()) {

                                }
                            }
                            $transaction_arr = array($transaction, $transaction1);

                            //update repayment schedule
                            $ii = 1;
                            $repayments = RepaymentSchedule::where('loan_id', $loan_id)->get();
                            foreach ($repayments as $rep) {
                                $udate = Request::input('repayment_date');
                                RepaymentSchedule::where('id', $rep->id)->update(array('schedule_date' => $udate[$ii]));
                                $ii++;
                            }

                            //journal\
                            $branch_code = Request::input('branch_code');
                            $entry_date = date('Y-m-d H:i:s');
                            $invoice_number = Request::input('invoice_number');
                            $contract_id = Request::input('contract_id');

                            for ($m = 0; $m < count(Request::input('debit')); $m++) {
                                $description = Request::input('description')[$m];
                                $transaction_id = $transaction_arr[$m]->id;

                                $journal = new JournalRequiry;
                                $journal->tran_id = $transaction_id;
                                $journal->entry_date = $entry_date;
                                $journal->invoice_number = $invoice_number;
                                $journal->description = $description;
                                $journal->user_id = $loan->user_id;

                                if ($journal->save()) {
                                    $tran = TransactionsRequiry::where('id', '=', $transaction_id)->where('flag', '=', 0)->first();
                                    if (!empty($tran)) {
                                        $tran->flag = 1;
                                        $tran->save();
                                    }
                                    for ($k = 0; $k < 2; $k++) { // 0 = debit, 1 = credit
                                        $parent_debit = Request::input('parent_debit')[$m];
                                        $parent_credit = Request::input('parent_credit')[$m];
                                        $debit = Request::input('debit')[$m];
                                        $credit = Request::input('credit')[$m];
                                        $d_description = Request::input('d_description')[$m];
                                        $c_description = Request::input('c_description')[$m];

                                        $jd = new JournalDetail;
                                        $jd->journal_id = $journal->id;
                                        $jd->coa_id = $k == 0 ? $parent_debit : $parent_credit;
                                        $jd->reference = $contract_id;
                                        $jd->branch_code = $branch_code;

                                        $prev_bl = array_fill(0, 2, 0.0);
                                        $prev_row = JournalDetail::select('b_debit', 'b_credit')
                                            ->where('coa_id', $jd->coa_id)
                                            ->orderBy('id', 'desc')
                                            ->first();
                                        if (!empty($prev_row)) {
                                            $prev_bl[0] = $prev_row->b_debit;
                                            $prev_bl[1] = $prev_row->b_credit;
                                        }
                                        $jd->p_debit = $prev_bl[0];
                                        $jd->p_credit = $prev_bl[1];
                                        $jd->debit = $k == 0 ? $debit : 0;
                                        $jd->credit = $k == 1 ? $credit : 0;
                                        $jd->b_debit = floatval($prev_bl[0]) + floatval($jd->debit);
                                        $jd->b_credit = floatval($prev_bl[1]) + floatval($jd->credit);
                                        $jd->description = $k == 0 ? $d_description : $c_description;
                                        $jd->save();
                                    }
                                }
                            }  //END $m
                        }
                    }
                    return redirect()->route('loan_detail', [$loan_id]);
                }
            }
        }
        return redirect('/');
    }
    
    And this is my private method.

I just post my private method and i will delete their code and use my method instead.
    
      private function Initial_LoanData($loanId)
    {
        if (!empty($loanId)) {

            $loan = Loan::with('schedule')->where('id', '=', $loanId)->first();
            if(!empty($loan) && (int)$loan->holiday_flag == 1 && (int)$loan->status == 2) {

                    $loan->disburse_date = Request::input('disburse_on');
                    $loan->contract_date = Request::input('contract_date');
                    $loan->disburse_note = Request::input('note');
                    $loan->start_date = Request::input('schedule_date');
                    $loan->disburse_byuserid = Auth::user()->id;
                    return $loan;
            } else {
                return false; //If LoadId not match
            }
        } else {
            return false;
        }
    }
    private function Initial_Repayment($loan){

        if(is_object($loan)) {

            $repayment_array = LoanCalculate::monthly_loan_schedule(
                $loan->repayment_type, $loan->start_date, $loan->loan_duration,
                $loan->loan_amount,$loan->interest_rate, $loan->balloon,
                $loan->balloon_month,$loan->monthly_payment, $loan->balloon_amount_array,
                $loan->custom_flag, $loan->days_of_month, $loan->holiday_flag, $loan->holiday, $loan->schedule
            )[0];
            if(!empty($repayment_array)) {
                $total_days = 0;
                $total_interest = 0;
                $total_principal = 0;
                $total_monthly = 0;
                $total_principal_bal = 0;

                for ($i = 1; $i < count($repayment_array); $i++) {

                    $total_days = $total_days + $repayment_array[$i][1];
                    $total_interest += $repayment_array[$i][2];
                    $total_principal += $repayment_array[$i][3];
                    $total_monthly += $repayment_array[$i][4];
                    $total_principal_bal += $repayment_array[$i][5];
                }
                $average_bal = $total_principal_bal / $loan->loan_duration;
                return [
                    'repay'=>$repayment_array,
                    'total'=>[
                        'total_days'=>$total_days,
                        'total_interest'=>$total_interest,
                        'total_principal'=>$total_principal,
                        'total_monthly'=>$total_monthly,
                        'total_principal_bal'=>$total_principal_bal,
                        'average_bal'=>$average_bal
                    ]
                ];
            }
        }
        return false;
    }
    private function Save_RepaymentSchedule($repayment_array, $loanid){

        if(!empty($repayment_array)){

            for ($i = 1; $i < count($repayment_array); $i++) {
                $data = [
                    'loan_id' => $loanid,
                    'schedule_date' => $repayment_array[$i][0],
                    'date_num' => $repayment_array[$i][1],
                    'interest' => $repayment_array[$i][2],
                    'principal' => $repayment_array[$i][3],
                    'intraday_rate' => $repayment_array[$i][6]];
                return RepaymentSchedule::insert($data);
            }
        } else {
            return false;
        }
    }
    private function Save_Transaction($loan, $loanId) {

        $transaction = new TransactionsRequiry();
        $transaction->id = TransactionsRequiry::select('id')->orderBy('id', 'desc')->first()->id + 1;
        $transaction->loan_id = $loanId;
        $transaction->trans_date = date('Y-m-d');
        $transaction->trans_type = "Disbursement";
        $transaction->amount = $loan->loan_amount;
        $transaction->description = $loan->disburse_note;
        $transaction->balance = $loan->loan_amount;
        $transaction->user_id = $loan->user_id;
        return $transaction->save();
    }
    private function Save_LoanAccount($loan){

        $loan_acc = ClientLoanAccounts::select(['id', 'activated_on', 'status', 'account_name', 'account_no'])->where('id', '=', $loan->loan_account_id)->first();
        if (!empty($loan_acc) && count($loan_acc) > 0) {

            $loan_acc->activated_on = date("Y-m-d");
            $loan_acc->status = 2; // Activate(Std)
            $loan_acc->balance = $loan->loan_amount;
            $loan_acc->save();
        }
    }
    private function Save_TransactionWithFee_Charge ($transaction, $loan, $loanId){

        $return_Val = [];
        $fee_charge = new FeeCharge();
        $fee_charge->charge_type = 4; // upfront charge
        $fee_charge->charge_amount = floatval(Request::input('commission_fee')) * $transaction->amount / 100.0;
        $fee_charge->charge_date = date('Y-m-d');
        $fee_charge->receipt = Request::input('invoice_number');
        $fee_charge->note = "Upfront charge for " . $loan->contract_id;
        $return_Val['feeCharge'] = $fee_charge->save();
        if($return_Val['feeCharge'] != false) {

            // Transaction
            $transaction1 = new TransactionsRequiry();
            $transaction1->id = TransactionsRequiry::select('id')->orderBy('id', 'desc')->first()->id + 1;
            $transaction1->loan_id = $loanId;
            $transaction1->trans_date = date('Y-m-d');
            $transaction1->trans_type = "Fee Charge Repayment";
            $transaction1->amount = floatval(Request::input('commission_fee')) * $transaction->amount / 100.0;
            $transaction1->fee = $transaction1->amount;
            $transaction1->balance = $loan->loan_amount;
            $transaction1->description = $fee_charge->note;
            $transaction1->user_id = $loan->user_id;
            $return_Val['trans'] = $transaction1->save();

            if(count($return_Val) > 0 && $return_Val['trans'] != false){

                return $return_Val;
            }else{

                return $return_Val;
            }
        }else{
            return false;
        }
    }
    private function Save_Journal($transaction_id, $description, $loan, $entry_date, $invoice_number){

        $journal = new JournalRequiry;
        $journal->tran_id = $transaction_id;
        $journal->entry_date = $entry_date;
        $journal->invoice_number = $invoice_number;
        $journal->description = $description;
        $journal->user_id = $loan->user_id;
        return $journal->save();
    }
    private function Save_journal_Detail($m, $journal, $contract_id, $branch_code){

        for ($k = 0; $k < 2; $k++) { // 0 = debit, 1 = credit
            $parent_debit = Request::input('parent_debit')[$m];
            $parent_credit = Request::input('parent_credit')[$m];
            $debit = Request::input('debit')[$m];
            $credit = Request::input('credit')[$m];
            $d_description = Request::input('d_description')[$m];
            $c_description = Request::input('c_description')[$m];

            $jd = new JournalDetail;
            $jd->journal_id = $journal->id;
            $jd->coa_id = $k == 0 ? $parent_debit : $parent_credit;
            $jd->reference = $contract_id;
            $jd->branch_code = $branch_code;

            $prev_bl = array_fill(0, 2, 0.0);
            $prev_row = JournalDetail::select('b_debit', 'b_credit')->where('coa_id', $jd->coa_id)->orderBy('id', 'desc')->first();
            if (!empty($prev_row)) {
                $prev_bl[0] = $prev_row->b_debit;
                $prev_bl[1] = $prev_row->b_credit;
            }
            $jd->p_debit = $prev_bl[0];
            $jd->p_credit = $prev_bl[1];
            $jd->debit = $k == 0 ? $debit : 0;
            $jd->credit = $k == 1 ? $credit : 0;
            $jd->b_debit = floatval($prev_bl[0]) + floatval($jd->debit);
            $jd->b_credit = floatval($prev_bl[1]) + floatval($jd->credit);
            $jd->description = $k == 0 ? $d_description : $c_description;
            return $jd->save();
        }
    }
